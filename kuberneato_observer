#!/bin/bash
function show_help
{
    echo "Usage: kubernato_observer (-options) <app_name>"
    echo ""
    echo "  <app_name> is the application name (from app.kubernetes.io/name label)"
    echo "  <name_name> is only required if there is more than one node"
    echo "              running in the pod."
    echo ""
    echo "Options:"
    echo ""
    echo "  -c <cookie>       Set the cookie value (if not auto-detected)"
    echo "  -n <namespace>    Kubernetes namespace (if not auto-detected)"
    echo ""
    echo "Example:"
    echo "  kubernato_observer your-service"
    echo "  kubernato_observer -c thecookievalue your-service"
    echo ""
    exit
}

if [[ "$1" == "" ]]; then
  show_help
fi

OPTIND=1
namespace=""

while getopts "?c:n:" opt; do
    case "$opt" in
    \?)
        show_help
        exit 0
        ;;
    c)  COOKIE=$OPTARG
        ;;
    n)  namespace=$OPTARG
        ;;
    esac
done

shift $((OPTIND-1))

[ "${1:-}" = "--" ] && shift

app_name=$1
echo "Looking for app: $app_name"

function get_pod_info() {
    local app=$1
    local ns=$2
    local ns_flag=""
    if [[ -n "$ns" ]]; then
        ns_flag="-n $ns"
    else
        ns_flag="-A"
    fi
    kubectl get pod $ns_flag \
        -l=app.kubernetes.io/name=$app,app.kubernetes.io/component!=database \
        --field-selector=status.phase==Running \
        -o jsonpath="{.items[0]['metadata.name', 'metadata.namespace']}" 2>/dev/null
}

pod_info=$(get_pod_info "$app_name" "$namespace")
if [[ -z "$pod_info" ]]; then
    echo "Error: No running pod found for app '$app_name'"
    if [[ -n "$namespace" ]]; then
        echo "Available pods in namespace '$namespace':"
        kubectl get pods -n "$namespace" -l app.kubernetes.io/name="$app_name"
    else
        echo "Available pods across all namespaces:"
        kubectl get pods -A -l app.kubernetes.io/name="$app_name"
    fi
    exit 1
fi

# Split pod_info into name and namespace
read -r pod_name namespace <<< "$pod_info"
echo "Found pod: $pod_name in namespace: $namespace"

# Get ERTS version from pod
echo "Getting ERTS version..."
folders=$(kubectl exec -n $namespace $pod_name -- ls "/app")
erts_folder=$(echo "$folders" | grep -o "erts-[0-9\.]*")

if [[ -z "$erts_folder" ]]; then
    echo "Could not find ERTS folder in /app"
    exit 1
fi

# Get Erlang node info directly from the pod's epmd
echo "Getting Erlang distribution port..."
port_app=$(kubectl exec -n $namespace $pod_name -- /app/$erts_folder/bin/epmd -names | tail -1)
echo "Debug: Raw epmd output from pod: $port_app"

# Then we use that data to parse the application name (ie banana-service is technically banana). Note
# that this and the port below have some weird formatting issues and new lines so we do some string
# replacement magic to ensure it works with iex.
erlang_app=$(echo "$port_app" | tail -1 | cut -d" " -f 2 | tr '\n' ' ' | sed -e 's/[^a-zA-Z0-9\-_]/ /g' -e 's/^ *//g' -e 's/ *$//g' | tr -s ' ')

# Parse the application name and port
erlang_port=$(echo "$port_app" | sed -nE 's/.*at port ([0-9]+).*/\1/p')

if [[ -z "$erlang_port" ]]; then
    echo "Could not determine Erlang distribution port"
    exit 1
fi

echo "Found Erlang distribution port: $erlang_port"

# Get the actual node name from the running application
echo "Getting actual node name from application..."
node_output=$(kubectl exec -n $namespace $pod_name -- /app/bin/$erlang_app rpc "Node.self() |> to_string |> IO.puts")
node_name=$(echo "$node_output" | tr -d "'")
echo "Debug: Got node name from application: $node_name"

# Check if node_name is empty
if [[ -z "$node_name" ]]; then
    echo "Error: Could not retrieve node name from the application."
    exit 1
fi

# Validate node_name format
if ! [[ "$node_name" =~ ^[a-z0-9_]+@[a-zA-Z0-9.-]+$ ]]; then
    echo "Error: Invalid node name format: '$node_name'. Expected format: <name>@<host>"
    exit 1
fi

# Get pod's IP address
pod_ip=$(kubectl get pod -n $namespace $pod_name -o jsonpath='{.status.podIP}')
echo "Pod IP: $pod_ip"

## Detecting the secret distribution cookie
if [[ "$COOKIE" == "" ]]; then
    echo "Trying to load cookie from pod..."
    raw_cookie=$(kubectl exec -n $namespace $pod_name -- cat /app/releases/COOKIE)
    COOKIE=$(echo "$raw_cookie" | tr '\n' ' ' | sed -e 's/[^a-zA-Z0-9=_]/ /g' -e 's/^ *//g' -e 's/ *$//g')
fi

if [[ -z "$COOKIE" ]]; then
    echo "Could not determine the cookie"
    exit 1
fi

echo "Cookie: ${COOKIE}"

whoami=$(whoami | sed 's/[^a-zA-Z0-9]//g' | tr '[:upper:]' '[:lower:]')
local_name="remsh_${whoami}_$$"

# Set up port forwarding and inet config
echo "Setting up networking..."
tmp_dir=$(mktemp -d)

function cleanup {
    echo $1
    if [ -f "$tmp_dir/portforward.pid" ]; then
        kill $(cat $tmp_dir/portforward.pid) 2>/dev/null
    fi
    rm -rf $tmp_dir
    exit
}

# Set up Erlang inet configuration
export ERL_INETRC=$tmp_dir/erl_inetrc

# Extract the host part from the node name (after @)
remote_host=$(echo "$node_name" | cut -d@ -f2)

# Convert pod IP to tuple format (e.g., "10.128.5.12" -> "{10,128,5,12}")
ip_tuple=$(echo "$pod_ip" | sed 's/\./,/g')
ip_tuple="{$ip_tuple}"

cat > $ERL_INETRC << EOF
{host, {127,0,0,1}, ["$remote_host"]}.
{host, $ip_tuple, ["$remote_host"]}.
{lookup, [file, native]}.
EOF

kubectl port-forward pod/"$pod_name" -n "$namespace" 4369:4369 "$erlang_port:$erlang_port" &
port_forward_pid=$!

sleep 2  # Give port-forward time to establish

echo "Starting iex shell..."
echo "Once connected, run:"
echo "1. Node.connect(:'$node_name')"
echo "2. :observer.start()"
echo ""

# Start IEx with distribution configuration
iex --name "$local_name@$remote_host" \
    --cookie "$COOKIE" \
    --hidden

cleanup "Done"