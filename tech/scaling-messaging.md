# A year of scaling a messaging service at a Startup

>If you don't know me already, I work at a small startup Limechat. <br>
> We are building E-commerce on Chat.

The word chat in this context means anything that can be done via messages will be done via messages. To deal with these messages we have built a service that facilitates the sending of messages and receiving status updates and incoming messages from the users.

I want to break this post down into two parts
 - Scaling Challenges in Sending messages.
 - Scaling Challenges in Webhooks system.

<br>

> A webhook is an api request sent to us by whatsapp when a message is sent, delivered, read or failed. We use these webhooks to update the status of the message in our database and to relay this information to our internal systems like CRM, Bot etc.


## Scaling Challenges in Sending messages
We have an API that supports sending two types of messages, a message can be sent one by one or a single messages
can be sent in bulk to multiple users fecilitating things like broadcast on whatsapp.

You could aruge that just sending one by one would also have easily fecilitated the broadcast where the client can
just use that API multiple times but that would have been an inefficient design, we are wasting network resources
of the client and our server as well and on top of it client should also be scalable enough to send hundereds of thousands of API calls in a short span of time.

Our API looks something like this

**Example bulk API**

`api/v1/messages`

parameters:
 - type: [bulk, normal] default: normal

body:
  - fileURL: A CSV file contains list of phone numbers
  message: A JSON object containing the message content

Code for this is pretty simple
```js

async function sendMessageApi(request, reply) {

	const { type, fileURL, messageContent} = request.body;
	const job = Jobs.create({ type, fileURL, data });
	if (type === 'bulk') {
		await addJobToQueue('messages:bulk', 'send_bulk_messages', job.id);
	} else {
		await addJobToQueue('messages', 'send_message', job.id);
	}
	reply.send({ok: true});
}


async function sendMessageJob(jobId) {
	const job = await Jobs.findById(jobId);
	const { phone, message } = job.data;	
	// api call to whatsapp
}

async function sendBulkMessagesJob(jobId) {
	const jobData = await Jobs.findById(jobId);
	if (!jobData){
		logger.error('Job not found, id: %', jobId);
		return;
	}
	const phones = await fetchPhones(jobData.fileURL);
	phones.forEach(async function(phone){
		await sendMessageJob(phone, jobData.messageContent);
	});
}
```
 - We are using bullMQ for queuing the jobs in the background.
 - We are not sending the messages synchronously as it can overwhelm the main server.
 - Also our bulk API supports sending of 500k messages in a single request which would take a lot of time to process synchronously.


This would easily scale & we still have no problem with this type of architecture for single messages, but for bulk messages there is a lot of scope for improvement.

>**📢 Quick refresher on background processing** <br>
In web servers, synchronous processing handles requests and responds immediately, like Instagram's post API. However, APIs that require extensive processing, such as YouTube's video upload API, can't be handled this way.<br>
Background processing queues jobs and sends an "accepted" response to the client. The job is processed asynchronously, and the client may check its status later if the server provides an API for this purpose. This approach often uses distributed messaging queues like RabbitMQ, Kafka, or BullMQ. <br>
We're using BullMQ to do the same for our send message API.


Every Messaging Queue(MQ) out there have a basic property.
 - Atleast once delivery.
 - Even though bullmq promises exactly once delivery but that too fails and in the worst case give at least once delivery.

Meaning when we queue a job in the background messaging queue(MQ or bullMQ) promises it to be delivered at least once. If for some reason it doesn't receives acknowledgement from the consumer(subscriber) that it has received the message or it has processed the message, MQ will re-deliver this message. Now if our consumer is not idempotent this re-delivery can cause a disastrous effect which it did in our case.

```js

async function sendBulkMessagesJob(jobId: string): Promise<void> {

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

`sendBulkMessagesJob` function is called for each job that is queued when a bulk API call is received. Its working is pretty simple
 - Find the job in database.
 - Fetch phone numbers from CSV URL
 - send message to each phone number.

If for some reason re-delivery happens here we will again just fetch the jobData from database and send messages to all the phone numbers again, resulting in duplicate messages being sent to users, double the API calls made to whatsapp meaning more cost to us and also more resources wasted by the server. This problem will only become more sever with more re-deliveries.

The more the phone numbers, more are the chances of this problem to occur as large list of phone numbers will require more time, resulting in timeout meaning broker will assume it's not processed so re-delivery again and this goes on and on forever.

First thing we can do to solve this is configuring bigger timeout for this job.

```js
await addJobToQueue('messages:bulk', 'send_bulk_messages', data, { timeout: 300 }) // 300 seconds
```

But this only solves for the timeout re-delivery problem that too assuming it finishes under 300 seconds.
If for some reason re-delivery happened after 300 seconds we will still be sitting ducks.

**Idempotency**
An operation is idempotent if it has the same result no matter how many times it's applied.

To make our sendBulkMessagesJob idempotent we can introduce a parameter in our jobData document or relation `isProcessed`, we will mark this value as `true` once all the messages are sent and at the start of job we will check if this value is true or false if it's true we will terminate the job.

```js
async function sendBulkMessagesJob(jobId: string): Promise<void> {
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

As you can see, I have tried to demonstrate in above sorta visual. As processing of this job takes a lot of time... re-delivery can happen in the middle the execution of this job as well. To solve this issue we need to make sure that at a time there is only execution sendBulkMessagesJob happening, we can introduce locks to solve for it. We can employ redis to handle the locking part.

```js
async function sendBulkMessagesJob(jobId: string): Promise<void> {
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

## Beginner's guide to webhooks system 

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
			);
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
	const subStatus = await redis.get(`webhook:sub:${subId}:status`);
	return subStatus;
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
			);
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
