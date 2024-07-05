# A year of scaling a messaging service at a Startup

>If you don't know me already, I work at a small startup Limechat. <br>
> We are building commerce on Chat.

The word chat in this context means anything that can be done via messages will be done via messages. To deal with these messages we have built a service that facilitates the sending of messages and receiving status updates and incoming messages from the users.

We call this service UMS internally, it stands for Universal Messaging Service and that's is how I am going to refer to it as in this post.

UMS does two things
 - Sends messages to users
 - Receives messages/message updates from users

 For sending the message it has two APIs one for sending a single message and other for sending bulk messages. Sending bulk messages API facilitates the broadcast feature. We support broadcast of 500k messages in a single API call. And we have 100s of clients using this feature everyday. Currenly we send more than 1 million broadcast messages in a day at peak load. This combined with the single messages and incoming messages, UMS processes more than 10 million messages in a week and it is growing.

## Challenges faced in scaling bulk messages
With the API supporting 500k messages in a single call, the biggest challenge we have faced so far is duplication.
For very big broadcast like with around 100-200k messages, we would face issues like entire broadcast getting duplicated or sometimes all the messages in broadcast would get sent twice or even thrice.


**Example bulk API**

`api/v1/messages?type=bulk`

body:
- fileURL: A CSV file contains list of phone numbers
- message: A JSON object containing the message content

Code for this is pretty simple
```js

async function sendBulkMessagesApi(request, reply) {

	const { message, fileURL} = request.body;
	const broadcastJob = await Jobs.create({message, fileURL});  // create object in database
	await addJobToQueue('messages:bulk', 'send_bulk_messages', broadcast.id);
	reply.send({ok: true});
}


async function sendMessage(phone, message) {
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
		await sendMessageJob(phone, jobData.message);
	});
}

```
 - We are using bullMQ for queuing the jobs in the background.
 - We are not sending the messages synchronously as processing 500k messages synchronously can overwhelm the main server.


>**📢 Quick refresher on background processing** <br>
In web servers, synchronous processing handles requests and responds immediately, like Instagram's create post API. However, APIs that require extensive processing, such as YouTube's video upload API, can't be handled this way.<br>
Background processing queues jobs and sends an "accepted" response to the client. The job is processed asynchronously, and the client may check its status later if the server provides an API for this purpose. This approach often uses distributed messaging queues like RabbitMQ, Kafka, or BullMQ. <br>
We're using BullMQ to do the same for our send message API.


Every Messaging Queue(MQ) out there have a basic property.
 - Atleast once delivery.
 - Even though bullmq promises exactly once delivery but that too fails and in the worst case give at least once delivery.

Meaning when we queue a job in the background messaging queue(MQ or bullMQ) promises it to be delivered to the consumer at least once. If for some reason it doesn't receives acknowledgement from the consumer that it has received the message or it has processed the message, MQ will re-deliver this message. Now if our consumer is not idempotent this re-delivery can cause a disastrous effect which it did in our case.

```js

async function sendBulkMessagesJob(jobId: string): Promise<void> {

	const jobData = await Jobs.findById(jobId);
	if (!jobData){
		logger.error('Job not found, id: %', jobId);
		return;
	}
	const phones = await fetchPhones(jobData.fileURL);
	phones.forEach(async function(phone){
		sendMessage(phone, jobData.message);
	})
}
```

`sendBulkMessagesJob` function is called for each job that is queued when a bulk API call is received. Its working is pretty simple
 - Find the job in database.
 - Fetch phone numbers from CSV URL
 - send message to each phone number.

If for some reason re-delivery happens here we will again just fetch the jobData from database and send messages to all the phone numbers again, resulting in duplicate messages being sent to users, double the API calls made to whatsapp meaning more cost to us and also more resources wasted by the server. This problem will only become more sever with more re-deliveries.

The more the phone numbers, more are the chances of this problem to occur as large list of phone numbers will require more time to process, resulting in timeout meaning broker will assume it's not processed so re-delivery again and this goes on and on forever.

First thing we can do to solve this is configuring bigger timeout for this job.

```js
await addJobToQueue('messages:bulk', 'send_bulk_messages', jobId, { timeout: 300 }) // 300 seconds
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

 - let's say job A takes a lot of time in the fetchPhones or sendMessage section.
 - During this step re-delivery happened again, this time we still will pass isProcessed check and send duplicate messages and this could spiral again into another re-delivery.
 - To solve this we can use locks so at a time there is only one execution for the bulk messages job happening.

```js
async function sendBulkMessagesJob(jobId: string): Promise<void> {
	const lockHash = 'lock:bulk:' + jobId;
	try {	
		await redisLock(lockHash, 900); // lock for 900 seconds
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
			await sendMessage(phone, jobData.messageContent);
		});
		await updateJob(jobId, {...jobData, isProcessed: true});
	} finally {
		releaseLock(lockHash);
	}
}
```


With this we finally would be able to avoid all the problems of re-delivery. There is still margin for improvments here in terms of sending the messages.
 - we could use Promise.all() to send all the messages in one.
 - But this could also introduce problems like rate limiting from whatsapp servers.
 - To avoid that we send messages in batches like a batch, batch size can be configurable based on the rate limit of target api server.
 - We can also use things like async.mapLimit that let's us choose the limit of how many operations should happen at a time.



## Building a webhooks system for receiving incoming messages and status updates.

A webhook is a way of communicating b/w servers. There are two servers one is a publisher server that obviously publishes the messages (post API call) and other is a subscriber server(handles the post request) that subscribes to the webhooks.

In our messaging service we need webhooks system to relay the status updates (sent, delivered, read, failed) of messages to our internal other systems like CRM, bot. A CRM uses these webhooks to show the incoming messages or to update the status of the message etc.

At peak load UMS processes at least 120k webhooks requests in a minute and then sends the same to relevant internal and external servers.



**Building a webhooks server**  <br>
We can start simple, in our case when we get webhook from whatsapp we process it to a standard format, create/update the relevant database object, and relay the webhooks further downstream to our internal services (CRM, bot) and to public subscribers.

The flow looks like this <br>

Whatsapp ----> Messaging Service ---> Parse webhook--> Dispatch ---> CRM, Bot, Public Clients
<br>
We can start with something like below <br>

`https://messaging.company.com/v1/webhooks.handle-wa`

```js

async function(request, reply){

	const body = request.body;
	const parsedData = await parseWebhook(body);
	await StorageService.process(parsedData);
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
