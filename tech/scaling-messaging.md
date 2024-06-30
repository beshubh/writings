# A year of scaling a messaging service at a Startup

If you don't yet know I work at a small startup Limechat.

We are building E-commerce on Chat.

The word chat in this context means anything that can be done via messages will be done via messages. To deal with these messages we have built a service that facilitates the sending of messages and receiving status updates and incoming messages from the users.

These are just short anecdotes of how we went through it, it may help you in future.


**First principles coding** <br>
While writing the code for the service we just focused on the first principles didn't think too much about optimizations. We were majorly focused on only writing clean code. Every function should be highly cohesive, code should be decoupled, we focused on building the top level interfaces required by this service and these interfaces should be segregated enough that if we had to let's say add new integration (currently we support whatsapp only) let's say instagram we didn't have to touch any existing file for the most part. 

Sending of messages and processing of status updates via webhooks and then further sending those webhooks downstream to other services , all of this involves network IO, we rarely do any CPU bound operation here, so we chose NodeJS to power the backend of this service.


**Idempotency & Queuing systems**

This messaging service also supports sending of messages in bulk, a client can send as many as 500K messages at once using this bulk API.

Our first principles got us to the place where we were easily able to handle around 50k bulk messages with little to no issues even 100k wasn't a problem but we could see issues occurring with multiple client doing bulk API calls with 100k messages.

Example bulk API

`api/v1/messages`

parameters:
 - type: [bulk, normal] default: normal
body:
  - file_url: A CSV file contains list of phone numbers

Processing 500k phone numbers synchronously is not an option, so when we receive the bulk request we create an object in the database attaching the file_url and queue a job in the background using bullmq to process the CSV and send messages.

>**📢 Quick refresher on pub-sub**
A pub sub system consists of three components a publisher, subscriber and a broker. Publisher publishes the messages(don't confuse it with message we send to user on whatsapp) broker takes this messages and delivers it to the subscribers. This mechanism in our case is powered by bullmq that uses redis as a broker.

Basic property of these queuing systems 
 - At least once delivery

Meaning when we queue a job in the background messaging queue(MQ or bullMQ) promises it to be delivered at least once. If for some reason it doesn't receives acknowledgement from the consumer(subscriber) that it has received the message or it has processed the message, MQ will re-deliver this message. Now if our consumer is not idempotent this re-delivery can cause a disastrous effect which it did in our case.

```js

async function processBulkJob(jobId: string): Promise<void> {

	const jobData = await Jobs.findById(jobId);
	if (!jobData){
		logger.error('Job not found, id: %', jobId);
		return;
	}
	const phones = await fetchPhones(jobData.fileURL);
	phones.forEach(async function(phone){
		sendMessage(phone, jobData.messageContent);
	})
}
```

`processBulkJob` function is called for each job that is queued when a bulk API call is received. Its working is pretty simple
 - Find the job in database.
 - Fetch phone numbers from CSV URL
 - send message to each phone number.

If for some reason re-delivery happens here we will again just fetch the jobData from database and send messages to all the phone numbers again, resulting in duplicate messages being sent to users, double the API calls made to whatsapp meaning more cost to us and also more resources wasted by the server. This problem will only become more sever with more re-deliveries.

The more the phone numbers, more are the chances of this problem to occur as large list of phone numbers will require more time, resulting in timeout meaning broker will assume it's not processed so re-delivery again and this goes on and on forever.

First thing we can do to solve this is configuring bigger timeout for this job.

```js
await addJobToQueue('messages:bulk', 'process_bulk_job', data, { timeout: 300 }) // 300 seconds
```

But this only solves for the timeout re-delivery problem that too assuming it finishes under 300 seconds.
If for some reason re-delivery happened after 300 seconds we will still sitting ducks.

**Idempotency**
An operation is idempotent if it has the same result no matter how many times it's applied.

To make our processing idempotency we can introduce a parameter in our jobData document or relation `isProcessed`, we will mark this value as `true` once all the messages are sent and at the start of process job we will check if this value is true or false if it's true we will terminate the job.

```js
async function processBulkJob(jobId: string): Promise<void> {
	const jobData = await Jobs.findById(jobId);
	if (!jobData){
		logger.error('Job not found, id: %', jobId);
		return;
	}
	if (jobData.isProcessed){
		logger.debug('Job already processed %', jobId);
		return;
	}
	const phones = await fetchPhones(jobData.fileURL);
	phones.forEach(async function(phone){
		sendMessage(phone, jobData.messageContent);
	});
	await updateJob(jobId, {...jobData, isProcessed: true});
}
```



Okay so now we have solved re-delivery problem with timeout and idempotency, but we still have a problem here

--------time-----------------------------------------------------------> <br>

-> Queue-->processing--fetch_phones--sending-----sending------------>completed <br>
->------------------------re-delivery-->processing--fetch_phones-- <br>

As you can see, I have tried to demonstrate in above sorta visual. As processing of this job takes a lot of time... re-delivery can happen in the middle the execution of this job as well. To solve this issue we need to make sure that at a time there is only execution processBulkJob happening, we can introduce locks to solve for it. We can employ redis to handle the locking part.

```js
async function processBulkJob(jobId: string): Promise<void> {
	const lockHash = 'lock:bulk:' + jobId;
	try {	
		await redisLock(lockHash, 300); // lock for 300 seconds
		const jobData = await Jobs.findById(jobId);
		if (!jobData){
			logger.error('Job not found, id: %', jobId);
			return;
		}
		if (jobData.isProcessed){
			logger.debug('Job already processed %', jobId);
			return;
		}
		const phones = await fetchPhones(jobData.fileURL);
		phones.forEach(async function(phone){
			sendMessage(phone, jobData.messageContent);
		});
		await updateJob(jobId, {...jobData, isProcessed: true});
	} finally {
		releaseLock(lockHash);
	}
}
```


With this we finally would be able to avoid all the problems of re-delivery, there is still margin for improvements but those are very rare cases, we have almost never encountered any such cases as of now.

To solve for these rare cases though you can employ the idempotency at sendMessage level as well and only allow sending the messages that are still in pending state, as you will be updating the message status on each successful api call. <br>


**Beginner's guide to webhooks system**

A webhook is a way of communicating b/w servers. There are two server one is a publisher server that obviously publishes the messages and other is a subscriber server that subscribes to the webhooks.

In our messaging service we need webhooks system to relay the status updates (sent, delivered, read, failed) of messages to our internal other systems like CRM, bot. A CRM uses these webhooks to show the incoming messages or to update the status of the message etc.



**Building a webhooks server**  <br>
We can start simple, in our case we just relay the webhooks we receive at the messaging service server further downstream to our internal services (CRM, bot) and to public subscribers (some clients use this service for sending messages and receiving webhooks)

The flow looks like this

Whatsapp ----> Messaging Service ---> Parse webhook--> Dispatch ---> CRM, Bot, Public clients
<br>
We can start with something like below <br>

`https://messaging.company.com/v1/webhooks.handle-wa`

```js

async function(request, reply){

	const body = request.body;
	const parsedData = await parseWebhook(body);
	const webhookClients = await WebhookClients.findAll();
	webhookClients.forEach( function(client) {
		fetch('post', {
			url: client.url,
			body: parsedData
		})
	});
	reply.send({ok: true});
}
```

 - Now as influx of webhooks from whatsapp can be huge we should process these webhooks in the background.
- We can use bullMQ in our case to achieve this.

```js
async function(request, reply){

	const body = request.body;
	await addJobToQueue('webhooks', 'process_wa_webhook', body);
	reply.send(ok: true);
}


async fuction processWAWebhook(jobData) {
	const parsedData = await parseWebhook(body);
	const webhookClients = await WebhookClients.findAll();
	webhookClients.forEach( function(client) {
		fetch('post', {
			url: client.url,
			body: parsedData
		})
	});
}
```


- As webhook is just an API call it can fail as well, may be subscriber server is down or can't process the requests at the time we should introduce retry as well.
- For this we segregate the sending of the actual webhook request in a separate job itself.


```js


const WEBHOOK_MAX_RETRIES = 3;
const BACK_OFF_MULTIPLIER = 2;
const FAILURES_THRESHOLD = 20;

async fuction processWAWebhook(jobData) {
	const parsedData = await parseWebhook(body);
	const webhookClients = await WebhookClients.findAll();
	webhookClients.forEach( async function(client) {
		await addJobToQueue(
		'webhook', 
		'dispatch_webhooks',
		{ url: client.url, parsedData, }
		);
	});
}

async function disPatchWebhook(jobData) {
	const { delay = 1000, retries = 0 } = jobData;
	
	try {
			const res = await fetch('post', {
				url: jobData.url,
				body: jobData.parsedData,
				timeout: 5000 // configuring request timeout for 5 seconds
			});
	} catch (error) {
		if (retries < WEBHOOK_MAX_RETRIES) {
			const nextDelay = delay * BACK_OFF_MULTIPLIER;
			await addJobToQueue(
			'webhook',
			'dispatch_webhook',
			{...jobData, retries: retries + 1, delay: nextDelay},
			{ delay: delay} // delay the job by delay milliseconds.
			)
		}
	}
}
```

 - With this we have handled the cases a webhook subscriber might be down.
 - we have also added timeout in the request this will help us avoiding any hung network connections or slow subscribers and we can convey this information to the subscribers that you have to respond with 5 seconds or receive a webhook.

**Queue Segregation**

 - With can improve our design by segregating the queues for normal webhooks and the webhooks being retried.
 - Because the normal webhooks should always have the higher priority that the webhooks that are being retried which might not be the case with our current design.

```js
async function disPatchWebhook(jobData) {
	const { delay = 1000, retries = 0 } = jobData;
	
	try {
			const res = await fetch('post', {
				url: jobData.url,
				body: jobData.parsedData,
				timeout: 5000 // configuring request timeout for 5 seconds
			});
	} catch (error) {
		if (retries < WEBHOOK_MAX_RETRIES) {
			const nextDelay = delay * BACK_OFF_MULTIPLIER;
			await addJobToQueue(
			'webhook_dlq',
			'dispatch_webhook',
			{...jobData, retries: retries + 1, delay: nextDelay},
			{ delay: delay} // delay the job by delay milliseconds.
			)
		}
	}
}
```

 - This will  queue the retry webhooks in a different queue so we can scale dead letter queue (DLQ) worker and main webhooks worker independently.
 - Remaining problem now is retrying indefinitely when a subscriber is just not responding at all which can end up wasting resources and also un-necessary scaling issues.
 - To avoid this issue we can tombstone a subscriber if there has been more than certain threshold failures on an endpoint in last 24 hours or whatever timeframe you want.
 - This can easily achieve via redis as well.

```js

const FAILURE_TIMEFRAME = 24 * 60 * 60 * 1000; // 24 hours
const FAILURES_THRESHOLD = 20;

export async function checkWebhookSubscriberStatus(subId: string) {
  const subscriptionStatus = await redis.get(`webhook:sub:${subId}:status`);
  return subscriptionStatus;
}

async function markWebhookSubscriberDead(subscriptionId: string) {
  await redis.set(`webhook:sub:${subId}:status`, 'dead');
}

export async function markWebhookSubscriberAlive(subId: string) {
  await redis.set(`webhook:sub:${subId}:status`, 'alive');
  await redis.del(`webhook:sub:${subId}:failures`);
}

async function incrementFailures(subId: string) {
  const now = Date.now();
  await redis.zadd(`webhook:sub:${subId}:failures`, now, now.toString());
  await redis.zremrangebyscore(
  `webhook:sub:${subId}:failures`, '-inf', now - FAILURE_TIMEFRAME);
  return redis.zcard(`webhook:sub:${subId}:failures`);
}

async function disPatchWebhook(jobData) {
	const { delay = 1000, retries = 0 } = jobData;
	const subStatus = await checkSubsriberStatus(subscriptionId);
	if (subscriptionStatus === 'dead') {
		logger.info(
		'Subscription is dead, not dispatching webhook event to %s', url
		);
		return;
	}
	try {
			const res = await fetch('post', {
				url: jobData.url,
				body: jobData.parsedData,
				timeout: 5000 // configuring request timeout for 5 seconds
			});
	} catch (error) {
		const failures = await incrementFailures(subscriptionId);
	    if (failures > FAILURES_THRESHOLD) {
	      await markWebhookSubscriberDead(subscriptionId);
	      return;
	    }
		if (retries < WEBHOOK_MAX_RETRIES) {
			const nextDelay = delay * BACK_OFF_MULTIPLIER;
			await addJobToQueue(
			'webhook_dlq',
			'dispatch_webhook',
			{...jobData, retries: retries + 1, delay: nextDelay},
			{ delay: delay} // delay the job by delay milliseconds.
			)
		}
	}
}
```
 - With this implementation we have successfully dealt with dead servers as well.
 - To further solidify the implementation you can send an email alert to your clients that their webhook server is not responsive and we won't be sending any more webhooks to their endpoint.
 - We can also give the clients an option in the email to resurrect the dead subscriber of theirs, so when they have fixed the issue they can subscribe again.

<br>
<br>
<br>
<br>
