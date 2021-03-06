# Serverlesss Eventing With PubSub

__Corresponding [Blog Post](https://thejaysmith.com/titles/serverlessjay/serverless-eventing/serverless-eventing-google-native-with-pubsub/ "Blog Post")__

[Knative Eventing](https://knative.dev/docs/eventing/) offers a variety of Event Sources to use when you are building a serverless eventing platform. So far, my blog has covered [SinkBinding](https://thejaysmith.com/titles/blogroll/serverless-eventing-sinkbinding-101/) as well as [Kafka](https://thejaysmith.com/titles/serverlessjay/serverless-eventing-modernizing-legacy-streaming-with-kafka/ "Kafka")and [NATS](https://thejaysmith.com/titles/serverlessjay/severless-eventing/serverless-eventing-cloud-native-messaging-with-nats-streaming-server/ "NATS") .

This tutorial will show you how to use [Google Cloud PubSub](https://cloud.google.com/pubsub "Google Cloud PubSub") as a Knative Eventing source. PubSub is Google Cloud's messaging system. It is truly serverless because you do not have to provision any brokers or workers. You simply create topics and consume. It is also global so you don't have to worry about provisioning in certain regions or load balancing. It is a great solution for someone looking to go truly serverless in their Serverless Eventing strategy.

## Concepts

Google provides us with an [Event Source](https://knative.dev/docs/eventing/sources/ "Event Source") called [CloudPubSubSource](https://github.com/google/knative-gcp/blob/master/docs/examples/cloudpubsubsource/README.md "CloudPubSubSource"). Event Sources are [Kubernetes Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/ "Kubernetes Custom Resources") that acts as a link between an event producer and the event sink. There are many types of event sources, some maintined by the Knative team and some maintained by third parties. You can even [create your own](https://knative.dev/docs/eventing/samples/writing-event-source/ "create your own")!

Google PubSub will be our Event Source. You will notice that we are using a different API than what we have used previously. For PubSub, Google created their own API that is KNative Eventing API compliant but also integrates some of Google Cloud's SDK, making integration with Google Cloud services as event sources easier. This API is `messaging.cloud.google.com/v1beta1` rather than `sources.knative.dev/v1beta1` or `messaging.knative.dev/v1alpha1`.

## Scenario

You are creating a financial services mobile app that allows users to trade in foreign currencies. It would be important to have the most up-to-date exchange information in order to make a decision. We want to collect the data and then store it somewhere where it can be pushed to individual user's mobile apps.

Because you have global users, you want to ensure global availability of the messages. From a scalability and reliability perspective, those messages should be sent to a messaging platform. This offers abstraction from your core applicaiton as your application is not communicating directly to the mobile front end.

Here we will create an application that regularly checks for currency exchange information between "USD" and "JPY". It will then send the information to an event sink which will write it to PubSub Topic.

## Setup Environment

First we will setup some basic environment variables. Be sure to replace `<your project>` with your actual Google Cloud project name.

```bash
export PROJECT_ID=<your project>
export PROJ_NUMBER=$(gcloud projects list --filter="${PROJECT_ID}" --format="value(PROJECT_NUMBER)")
export CLUSTER_NAME='cr-knative-pubsub'
export ZONE='us-central1-a'
export KO_DOCKER_REPO="gcr.io/${PROJECT_ID}"
```

And now let's get our GitHub Repo

```bash
git clone https://github.com/TheJaySmith/serverless-eventing
cd serverless-eventing
```

I have created a script called `setup-gke.sh` that simplifies the staging process. It will attempt to install [Google Cloud SDK](https://cloud.google.com/sdk/) if you don’t have it installed already. If it can execute `gcloud`, it skips this step. It will then do the following.

- Enable Google Cloud APIs if they aren’t already enabled
- Create a [GKE](https://cloud.google.com/kubernetes-engine) Cluster
- Setup [xip.io domain](https://cloud.google.com/run/docs/gke/default-domain) for GKE
- Install [Knative Eventing](https://knative.dev/docs/eventing/) and [Knative Monitoring](https://knative.dev/docs/serving/installing-logging-metrics-traces/)
- Give your compute service account access to Secret Manager

Now let's navigate back to the `serverless-eventing` directory and run the script.

```bash
chmod +x scripts/setup-gke.sh
sh scripts/setup-gke.sh
```

Alright, we are ready to get started.

## Setup AlphaVantage

For demos, [AlphaVantage](alphavantage.co "AlphaVantage") is my goto source. They have a free tier that allows around 500 API calls/day and it's easy to sign up. You can get your key [here](https://www.alphavantage.co/support/#api-key "here").

Some people have asked why I always recommend AlphaVantage when I do these demos. I will say that they pay me absolutely nothing to promote them. I just really like using their API for Serverless Eventing demos as financial data tends to be real-time in nature.

### Cloud Secret Manager

Google Cloud offers a [Cloud Secret Manager](https://cloud.google.com/secret-manager/) which gives you the ability to securely store your secrets encrypted in Google Cloud. Remember those four Twitter API keys we had earlier? We are going to store them in Google Cloud using the Secret Manager.

We will go from the Hamburger -> Security -> Secrets.

![secret manager](https://raw.githubusercontent.com/TheJaySmith/serverless-eventing/master/assets/images/secret-manager.png)

Let’s now Create a secret. We will name this secret `alpha-vantage-key` and give it the "Value" of the API Key you just created. You have now created a secured value that our applicaiton will use.

## Setting up PubSub

As mentioned earlier, one of the benefits of [PubSub](https://cloud.google.com/pubsub "PubSub") is that it is truly serverless. As the end user, you don't have to worry about provisioning brokers. You also pay for usage rather than paying for idle brokers. All of the infrastructure is abstracted so I can provision the service and create a topic simply by entering the following:

```bash
gcloud services enable pubsub.googleapis.com
gcloud pubsub topics create currency-pubsub
```

As you can see, there was no need to provision a broker or install an operator on cluster. Of course, the con here is that it's proprietary and currently not something that you can run in your own datacenter. Now if you are adopting a pure serverless model, it's fine. However, there are those people who do want the ability to control their technology, run it on premises, or avoid vendor lock-in by adopting open source software.

This is why we show a bunch of methods for using serverless eventing rather than push one product or another. It's best to find a tool that suits your project needs and developer culture.

One last thing, we need to install the PubSub component for Knative Eventing.

```bash
kubectl apply --filename https://github.com/google/knative-gcp/releases/download/v0.17.0/cloud-run-events.yaml
```

## PubSub Data Plane

For our application to authenticate with PubSub, we need to create a [Service Account](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform "Service Account"). We obviously don't want just any application writing or consuming our data.

First we'll create a service account called `knative-dataplane`

```bash
gcloud iam service-accounts create knative-dataplane
```

Next, we'll give the Service Account the proper permissions to access PubSub and as an Editor.

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:knative-dataplane@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/pubsub.editor
```

Finally, create a service account key, turn it into a Kubernetes secret, then delete the json file (for security reasons)

```bash
gcloud iam service-accounts keys create knative-dataplane.json \
--iam-account=knativedataplane@$PROJECT_ID.iam.gserviceaccount.com
kubectl --namespace default create secret generic google-cloud-key --from-file=key.json=knative-dataplane.json
rm knative-dataplane.json
```

## Building our Applications

Let's now build our application. First, let's make sure that `gcloud` will be properly authenitcated with the `docker` command. If you do not have Docker installed, you can find it [here](https://docs.docker.com/get-docker/ "here").

```bash
gcloud auth configure-docker
```

Next we will build our currency app. Let's go to the currency app folder.

```bash
cd tutorials/pubsub/app/currency/
```

Let's take a look at the app in the `currency.py` file.

```bash
CURR1 = 'USD'
CURR2 = 'JPY'
```

These are the currency values that we will be using. While I have hardcoded 'USD' and 'JPY', you can change this to anything that you want.

```bash
afx = ForeignExchange(key=ALPHAVANTAGE_KEY)

def make_msg(message):
    msg = '{"msg": "%s"}' % (message)
    return msgs


def get_currency():
    data, _ = afx.get_currency_exchange_rate(
            from_currency=CURR1, to_currency=CURR2)
    exrate = data['5. Exchange Rate']
    return exrate


while True:
    headers = {'Content-Type': 'application/cloudevents+json'}
    body = get_currency()
    requests.post(sink_url, data=json.dumps(body), headers=headers)
    time.sleep(30)
```

We first create an AlphaVantage object using our key called `afx`. The `make_msg` function formats the function. The `def_currency` function will use CURR1 and CURR2 and return an exchange rate. The while loop will execute the `def_currency` function, get the exchange rate, and send it to our event sink every 30 seconds. You could make it more or less but I chose '30' as it will give you more time to play with it during the 500 calls/day limit.

Now lets build the containers and push them to [Google Container Registry](https://cloud.google.com/container-registry "Google Container Registry").

```bash
docker build --build-arg PROJECT_ID=${PROJECT_ID} -t gcr.io/${PROJECT_ID}/pubsub-currency:v1  .
docker push gcr.io/${PROJECT_ID}/pubsub-currency:v1
```

Now we will build our `pubsub-client`. This application will receive the financial messages and send them to PubSub.

```bash
cd ../pubsub-client/
```

A lot is happening is this code so let's take a look at `pubsub-client.py`. For the sake of simplicity, we will focus on unique aspects of this code.

```bash
from google.cloud import pubsub_v1
import cloudevents.exceptions as cloud_exceptions
from cloudevents.http import from_http
```

Here we are importing some important libraries. We are importing the [PubSub Python Library](https://pypi.org/project/google-cloud-pubsub/ "PubSub Python Library") so that we can publish messages to the PubSub topic. We also import a library for [CloudEvents](https://cloudevents.io "CloudEvents").

We will be using [Flask](https://flask.palletsprojects.com/en/1.1.x/ "Flask") to handle the POST requests coming from our `natss-currency` application.

```bash
@app.route('/', methods=['POST'])
def default_route():
    if request.method == 'POST':
        content = request.data.decode('utf-8')
        info(f'Event Display received event: {content}')
        content = bytes(content, 'utf-8')
        future = publisher.publish(topic_path, data=content)


        return jsonify(hello=str(future))
    else:
        return jsonify('No Data')
```

This function will receive the financial data from our `pubsub-currency` service and then publish the event to PubSub.

Now let's build the containers.

```bash
docker build --build-arg PROJECT_ID=${PROJECT_ID} -t gcr.io/${PROJECT_ID}/pubsub-client:v1 .
docker push gcr.io/${PROJECT_ID}/pubsub-client:v1
```

Great, now it is time to test and deploy.

## Deploy and Use

First let's check out our manifests files.

```bash
cd ../../manifests
sed -i '' 's/PROJECT_ID/'${PROJECT_ID}'/g' pubsub-client.yaml
sed -i '' 's/PROJECT_ID/'${PROJECT_ID}'/g' pubsub-currency.yaml
```

First we will deploy a SinkBinding. Here you can see that we deploy the SinkBinding undo the name `pubsub-currency-sink-bind`. This will take the "subject" as the event source and the sink as the event sink. For these purposes we are using a Knative Service but [SinkBinding](https://knative.dev/docs/eventing/samples/sinkbinding/ "SinkBinding") does allow for you to use other Kubernetes objects such as datasets. I also have a demo [here](https://github.com/TheJaySmith/serverless-eventing/tree/master/tutorials/twitter-sink-binding, "here").

So let's up our binding.

```bash
kubectl apply -f pubsub-currency-sink-bind.yaml
```

Next we deploy the `pubsub-client` service. We want to ensure that our sink is ready to receive before we deploy the source. This service will act as the publisher to PubSub Topic.

```bash
kubectl apply -f pubsub-client.yaml
```

Finally we deploy the `pubsub-currency` service. This will start creating events as soon as we deploy.

```bash
kubectl apply -f pubsub-currency.yaml
```

All of our services are deployed so let's move on.

## Subscribe and Test

Let's subscribe and consume. First let's create an [Event Source](https://knative.dev/docs/eventing/sources/ "Event Source"). Take a look at `pubsub-source.yaml`.

```bash
apiVersion: events.cloud.google.com/v1
kind: CloudPubSubSource
metadata:
  name: pubsub-knative-source
spec:
  topic: currency-pubsub
  sink:
    ref:
      apiVersion: v1
      kind: Service
      name: pubsub-viewer
```

We are creating an Event Source  called `pubsub-knative-source` and it will subscribe to the PubSub topic named `currency-pubsub`. It will then send that data to be consumed by our Service named `pubsub-viewer`.

Let's apply this.

```bash
kubectl apply -f pubsub-source.yaml
```

Finally, we will deploy a service to read the events. In a real world application, this may be your service that processes the data in order to get an output. For our purposes, it'll just log the results. So let's go ahead and apply.

```bash
kubectl apply -f pubsub-viewer.yaml
```

Give it a few seconds then run this command

```bash
kubectl logs -l serving.knative.dev/service=pubsub-viewer -c user-container --since=10m
```

If all works, you should see something like this.

```bash
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: google.cloud.pubsub.topic.v1.messagePublished
  source: //pubsub.googleapis.com/projects/rikudog/topics/currency-pubsub
  id: 1533730805360920
  time: 2020-09-16T01:43:28.536Z
  datacontenttype: application/json
Data,
  {
    "subscription": "cre-src_default_pubsub-knative-source_8da47c4c-7e5b-45fe-a1fb-b4becda83ba1",
    "message": {
      "messageId": "1533730805360920",
      "data": "IjEwNS4zMDAwMDAwMCI=",
      "publishTime": "2020-09-16T01:43:28.536Z"
    }
  }
```

This is base64 encoded so we'll need to decode it. Fortunately, we can do this from the terminal.

```bash
$ echo IjEwNS4zMDAwMDAwMCI= | base64 --decode
"105.30000000"
```

Congrats! Your service is now consuming data in real time from PubSub!

## Summarize

In this demo, we used the Google Cloud PubSub to create a Knative Event Source and subscribe a consumer service ttot use the streaming data. The true serverless nature of PubSub makes it a great platform for Eventing Busses. While this only showed one service accepting data, you would be able to scale that by using multiple subscriptions, brokers, and/or triggers. In the future, I will show how to create PubSub as an Event Source.

## End

Let's cleanup our work. First, let's delete our PubSub Topic.

```bash
gcloud pubsub topics delete currency-pubsub
```

Next let's delete our Service Account.

```bash
gcloud iam service-accounts delete knative-dataplane
```

Be sure to delete your clusters!

```bash
gcloud beta container clusters create $CLUSTER_NAME --zone $ZONE
```
