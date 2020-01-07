# Datadog logs tutorial

## Chapter 2: Setup

Before we start collecting logs for our [Loggen](../README.md) application, we need to have it running somewhere. So, let's use [Blops](https://github.com/blacklane/blops) to setup:

- [x] Kubernetes integration
- [x] Drone integration
- [x] Datadog integration

### Generating the Blops artifacts

First of all, make sure you've the latest version of Blops installed in your localhost. You can find instructions on how to install and setup Blops [here](https://github.com/blacklane/blops#setup).

Once Blops is installed and condfigured, let's generate the artifacts:

```bash
$ blops generate --name # Don't need to change any other parameter, since it's a standalone app with no external access
```

Blops will generate two main resources:

- _deploy/_: a folder containing all the Kubernetes configuration for our app
- _.drone.yml_: the DroneCI YAML definition

Since our app is simple, we don't need some of the resources generated out of the box by Blops. So, we can delete the following files:

- _deploy/base/secrets.yaml_
- _deploy/base/service.yaml_

Also, make sure to remove any reference to these to files in _deploy/deployments.yaml_ file.

Next, let's setup our K8s deployment. Blops setup most of the stuff automatically, but we need to provide some special environment variables for our service. The variables define the frequency the logs are generated
and the error rate.

So, first, let's define these new variables in our overlay file:

```yaml
#
# deploy/overlays/production/env.yaml
#
# Generate logs every 250ms with an error rate of 5%
#
REPLICAS: 1
LOG_INTERVAL_IN_MS: "250"
ERROR_RATE: "0.05"
```

Next, let's define these variables in our K8s config resource:

```yaml
# deploy/base/config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loggen
  namespace: $NAMESPACE
  labels:
    app: loggen
data:
  LOG_INTERVAL_IN_MS: "$LOG_INTERVAL_IN_MS" # Use the value defined in the overlay
  ERROR_RATE: "$ERROR_RATE"                 # Use the value defined in the overlay
```

Finally, let's associate the config above with the loggen deployment:

```yaml
# deploy/base/loggen.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loggen
  namespace: $NAMESPACE
  labels:
    app: loggen
  annotations:
    blacklane.wait.timeout: 1000s
spec:
  replicas: $REPLICAS
  selector:
    matchLabels:
      app: loggen
  template:
    metadata:
      labels:
        app: loggen
    spec:
      dnsPolicy: Default
      containers:
      - name: loggen
        image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/loggen:$IMAGE_TAG
        envFrom:
        - configMapRef:
            name: loggen # <-- Associate the config with the deployment
```

This is everything you need to do related to Kubernetes. Now, let's move to Drone fine-tuning.

### Drone setup

Drone setup is defined in _.drone.yaml_, and we basically need to:

- [ ] Setup the `build` stage
- [ ] Remove unused stages

The build stage can be found in `pipeline/build`. In this stage, we will remove the `secret` section, since we won't use any Drone secret support and setup the stage to build our `Loggen` image using `Docker build`. Thus, our build section is the following:

```yaml
# .drone.yaml
#...
build:
  image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/drone-plugin-ecr-tagger-pusher:v3
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  commands:
    - $(aws ecr get-login --no-include-email --region eu-central-1)
    - docker build -t loggen:${DRONE_COMMIT_SHA} .
  when:
    event: push
    branch: master
#...
```

Next, let's remove some steps we don't need. These are `pipeline/test` and `pipeline/deploy_testing` (we're deploying loggen in production cluster since Datadog Logs is still not available in testing cluster).

After the adjustments, your _.drone.yml_ file should look similar to this one:

```yaml
# .drone.yml
pipeline:
  pull:
    image: yspro/drone-plugin-ecr-fetcher
    pull: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    images:
      - 721041513556.dkr.ecr.eu-central-1.amazonaws.com/kubernetes-deployment:v4
      - 721041513556.dkr.ecr.eu-central-1.amazonaws.com/drone-plugin-ecr-tagger-pusher:v3

  registry:
    image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/kubernetes-deployment:v4
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    commands:
      - blops registry init

  build:
    image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/drone-plugin-ecr-tagger-pusher:v3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    commands:
      - $(aws ecr get-login --no-include-email --region eu-central-1)
      - docker build -t loggen:${DRONE_COMMIT_SHA} .
    when:
      event: push
      branch: master

  push:
    image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/drone-plugin-ecr-tagger-pusher:v3
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ecr_repo_uri: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/loggen
    image_name: loggen:${DRONE_COMMIT_SHA}
    when:
      event: push
      branch: master

  deploy_production:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    image: 721041513556.dkr.ecr.eu-central-1.amazonaws.com/kubernetes-deployment:v4
    commands:
      - blops deploy production
    when:
      event: push
      branch: master

branches: [ master ]
```

K8s and Drone setup done, time to move to production.

### Build and deploy

Our app is ready for the cloud, so let's start the process of deploying it.

First of all, let's trigger a Drone build. This can be done by committing & pushing to the `master` branch. Once the push is done to master, head to [Drone repos dashboard](https://drone.blacklane.net/account/repos) and make sure to activate the build for `Loggen` app.

Wait the build finishes successfully. The Drone build we setup before makes sure to upload the `Loggen` image to `AWS ECR`, which is our official image repository.

Now, we can finally deploy the app to Kubernetes:

```bash
$ blops deploy production

# ... wait for the deploy to finish ...
$ AWS_PROFILE=production kubectl --context=production -n production get po -l app=loggen

# ... check the logs generated by the pod
$ AWS_PROFILE=production kubectl --context=production -n production log -l app=loggen
```

You should see your pod up and running and generating logs.

Now that we've logs being generated, let's move to the next chapter and see how to query and manipulate our logs.

[Go to next chapter](3-checking-the-logs.md)
