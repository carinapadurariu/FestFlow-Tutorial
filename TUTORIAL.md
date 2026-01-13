Implementing an Event-Driven Microservices Architecture with RabbitMQ and Kafka


Technologies: Python (FastAPI), RabbitMQ, Kafka, Docker


1. Introduction
In modern e-commerce systems (such as our FestFlow ticketing platform), handling high volumes of concurrent traffic is a critical challenge. If a user buys a ticket and the system attempts to process the payment, update the inventory, and send a confirmation email in a single synchronous (blocking) request, the server will likely fail under pressure.

The solution is an Event-Driven Architecture.

This tutorial explains how we decoupled order processing using two communication models:

Command Processing (RabbitMQ): To ingest orders quickly and guarantee their processing.

Event Streaming (Kafka): To notify downstream systems (Email, Analytics) that a sale has occurred, without blocking the main flow.

---------------------------------------------------------------------------------------------

2. System Architecture
Our system acts as a distributed system composed of 3 autonomous microservices, each running in its own Docker container:

Order Service (Producer): Receives the HTTP request from the client.

Inventory Service (Consumer & Producer): Processes the order and updates stock.

Email FaaS (Consumer): Reacts to events and sends notifications.

Data Flow: HTTP Request -> Order Service -> [RabbitMQ Queue] -> Inventory Service -> [Kafka Topic] -> Email Service


---------------------------------------------------------------------------------------------


3. Step 1: Decoupling the Order (RabbitMQ)
The first step is ensuring the user receives an instant response ("Order Received"), even if the actual processing takes time. For this, we use a Message Broker (RabbitMQ).

In the Order Service, we do not perform complex processing. We simply validate the data and push a JSON message into the orders queue.

Implementation (Order Service - Python):

Python

import pika
import json

# RabbitMQ Connection Configuration
connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='rabbitmq', port=5672)
)
channel = connection.channel()

# Declare the queue to ensure it exists
channel.queue_declare(queue='orders')

def send_order_to_queue(order_data):
    """
    Sends the order to the queue for asynchronous processing.
    The client does not wait for the processing to finish.
    """
    message = json.dumps(order_data)
    
    channel.basic_publish(
        exchange='',
        routing_key='orders',
        body=message
    )
    print(f" [x] Order sent to RabbitMQ: {message}")
    return True
Benefit: If the Inventory Service is temporarily down or overloaded, messages are not lost. They remain in the RabbitMQ queue until they can be processed (Durability).



---------------------------------------------------------------------------------------------

4. Step 2: Processing and Event Streaming (Kafka)
The Inventory Service is the "worker". It listens to the RabbitMQ queue, checks the stock, and decides if the ticket can be sold.

If the sale is successful, the Inventory Service becomes a Kafka Producer. It emits a TICKET_SOLD event. The major difference compared to RabbitMQ is that the message in Kafka can be consumed by multiple services that need it, not just one.

Implementation (Inventory Service):

Python

from kafka import KafkaProducer
import json

# Kafka Producer Configuration
producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda x: json.dumps(x).encode('utf-8')
)

def process_order(ch, method, properties, body):
    """
    Callback triggered automatically when a message arrives in RabbitMQ
    """
    order = json.loads(body)
    
    # ... Stock verification logic ...
    
    if stock_available:
        # Successful Sale -> Emit event to Kafka
        event = {
            "event_type": "TICKET_SOLD",
            "ticket_id": order['ticket_type'],
            "timestamp": time.time()
        }
        
        # Send to 'analytics_topic'
        producer.send('analytics_topic', value=event)
        print(f" [Kafka] Event emitted: {event}")
        
    # Acknowledge processing to RabbitMQ
    ch.basic_ack(delivery_tag=method.delivery_tag)


---------------------------------------------------------------------------------------------


5. Step 3: Reacting to Events (FaaS / Email Service)
The final component is a FaaS (Function as a Service) style microservice. It knows nothing about orders, stock, or payments. It knows only one thing: to listen to the Kafka topic and send an email when it sees a TICKET_SOLD event.

This model allows for easy addition of new features (e.g., an Analytics Service) without modifying existing code in Inventory or Order services.

Implementation (Email Service):

Python

from kafka import KafkaConsumer
import json

print(" [Email Service] Starting... Listening to 'analytics_topic'")

# Kafka Consumer Configuration
consumer = KafkaConsumer(
    'analytics_topic',
    bootstrap_servers=['kafka:9092'],
    auto_offset_reset='latest', # Listen only to new messages
    value_deserializer=lambda x: json.loads(x.decode('utf-8'))
)

# Main Event Loop
for message in consumer:
    event = message.value
    
    if event['event_type'] == 'TICKET_SOLD':
        print(f" [Email] Generating PDF for ticket: {event['ticket_id']}")
        # Simulate sending email
        send_email_to_user(event)


---------------------------------------------------------------------------------------------

6. Conclusion and Execution
This architecture offers three major advantages for the FestFlow project:

Scalability: We can process thousands of orders per second in the Order Service because RabbitMQ acts as a buffer.

Resilience: If the email service fails, orders are still processed. Emails will be sent later when the service recovers and reads from Kafka (Replayability).

Decoupling: Services do not directly depend on each other.

How to run the project: To start the entire infrastructure (including Zookeeper for Kafka), we use Docker Compose:

Bash

docker-compose up -d --build
