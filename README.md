# kuberneato_observer

This was inspired by https://github.com/dominicletz/remote_observe.

`kuberneato_observer` does the basic plumbing of setting up port forwarding via `kubectl` to an app
in one of your namespace(s) so that you can connect to it. Unlike `remote_observe`, we do not automatically
start `observer`, and instead set it so that you can connect to the node once networking has been configured.
The reason for this is that _sometimes_ `remsh` absolutely will not connect, but you can still likely do this
partially manual process.

## Installation

Mark the script `kubernato_observer` as executable and move it somewhere in your $PATH for convenience:

```
wget https://raw.githubusercontent.com/msutkowski/kuberneato_observer/main/kuberneato_observer
chmod +x ./kuberneato_observer
sudo mv ./kuberneato_observer /usr/local/bin
```

## Usage

```sh
kuberneato_observer some-app-name
# wait for the connection
Node.connect(:'your_beam_app@some.pod.ip') # this will be echo'd out
:observer.start()
# Once observer is opened, goto Nodes and select your_beam_app@some.pod.ip
```

Note: it will try to automatically resolve the cookie for the connection as well as a few other things.

### Requirements

In your application(s), you need ensure that in your mix.exs you've added the runtime_tools in your deployment:

```
  # Run "mix help compile.app" to learn about applications.
  def application do
    [
      mod: {My.Application, []},
      extra_applications: [:runtime_tools]
    ]
  end
```

## CLI Overview

```
Usage: kubernato_observe (-options) <app_name> (<node_name>)

  <app_name> is the application name (from app.kubernetes.io/name label)
  <name_name> is only required if there is more than one node
              running in the pod.

Options:

  -c <cookie>       Set the cookie value (if not auto-detected)
  -n <namespace>    Kubernetes namespace (if not auto-detected)

Example:
  kubernato_observe -c your-service
  kubernato_observe -c thecookievalue your-service
```
