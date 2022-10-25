# Queue

## Elements of a Message Queue Framework
1. **Sender/Publisher**: The element of MQF that publishes messages and sends them to an exchange.
2. **Message Queue Broker**: A broker is present between a publisher and a consumer to perfom communication between these two elements.
3. **Queue**: The job of a queue is to store messages.
4. **Receiver/Consumer**: The primary responsibility of consumers is to receive messages from an exchange and detemine which queue is ready to consume data

## Install RabbitMQ
1. To isntall RabbitMQ on your Ubuntu/Debian Server, use this command 
    ```
    sudo apt install -y rabbitmq-server
    ```
2. Install `amqp` php extenstion

## Setup Magento 2 for RabbitMQ
1. For connecting Magento 2 and RabbitMQ you need to setup queue connection in your `app/etc/env.php` file and add below section into that file


    replace `RABBITMQ_USERNAME` and `RABBITMQ_PASSWORD` in this sample!

    ```php
    return [
        ...
        'queue' => [
            'amqp' => [
                'host' => 'localhost',
                'port' => 5672,
                'user' => 'RABBITMQ_USERNAME',
                'password' => 'RABBITMQ_PASSWORD',
                'virtualhost' => '/'
            ]
        ],
        ...
    ];
    ```

2. Run `bin/magento setup:upgrade` in your project root directory
3. Run `bin/magento setup:di:compile` in your project root directory


## Create Queue System for Modules
1. Create Communication.xml
create `communication.xml` file in your module at `<vendor>/<module>/etc`

    ```xml
    <?xml version="1.0"?>
    <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Communication/etc/communication.xsd">
        <topic name="amooati_queue.queue.order" request="string" response="string">
        </topic>
    </config>
    ```
    **request** key is input datatype, it could be a class (dto) and **response** key is output datatype.

2. Create `queue_publisher.xml` in `<vendor>/<module>/etv`

    ```xml
    <?xml version="1.0"?>
        <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/publisher.xsd">
        <publisher topic="amooati_queue.queue.order">
            <connection name="amqp" exchange="amooati-order-exchange"/>
        </publisher>
    </config>
    ```

    **topic** must be same as the `topic.name` in `communication.xml` 

3. Create Publisher class under the `Vendor\Module\Publisher`:
    ```php
    <?php

    namespace Vendor\Module\Publisher;

    use Magento\Framework\MessageQueue\PublisherInterface;

    class Publisher
    {
        const TOPIC_NAME = 'amooati_queue.queue.order';

        private PublisherInterface $publisher;

        public function __construct(PublisherInterface $publisher)
        {
            $this->publisher = $publisher;
        }

        /**
        * @param array $data
        * @return mixed|null
        */
        public function publish(array $data)
        {
            return $this->publisher->publish(self::TOPIC_NAME, json_encode($data));
        }
    }
    ```

    topic name must be same as `topic.name` where we used in `communication.xml`.
    We use array and json_encode because we said that the input datatype is string!

4. Later you will use this publisher when you want to send a message.
    
    Example of usage:
    ```php
    $data = [
        'amoo' => 123
    ];
    $publisher = $this->objectManager->create(\Vendor\Module\Publisher\Publisher::class);
    $publisher->publish($data);
    ```

5. Create `queue_topology` in `<vendor>/<module>/etc`

    It will sends a message from publisher to queue

    ```xml
    <?xml version="1.0"?>
    <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/topology.xsd">
        <exchange name="amooati-order-exchange" type="topic" connection="amqp">
            <binding id="AmooAtiOrder" topic="cac.mobile_notification.notification" destinationType="queue"
                    destination="AmooAti-Queue"/>

        </exchange>
    </config>
    ```

    The exchanger name must be same as connection.exchange we used in `queue_publisher`

    topic name must be same as the one we used in `communication.xml`

6. Create `queue_consumer.xml` in `<vendor>/<module>/etc`
   ```xml
   <?xml version="1.0"?>
    <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="urn:magento:framework-message-queue:etc/consumer.xsd">
        <consumer name="amooati.order.consumer.one" queue="AmooAti-Queue" connection="amqp" handler="Vendor\Module\Consumer\Consumer::consumerProcess"/>
    </config>
   ```
    `consumer.queue` must be same as the destination we used in `queue_topology.xml`
    `handler` is the class and method of that class where handles messages from queue and process them

7. Create Consumer class under the `Vendor\Module\Consumer\ConsumerClass`

    ```php
    <?php

    namespace Vendor\Module\Consumer;

    use GuzzleHttp\Client as GuzzleClient;
    use GuzzleHttp\Exception\GuzzleException;
    use Psr\Log\LoggerInterface;

    class Consumer
    {
        protected LoggerInterface $logger;

        public function __construct(LoggerInterface $logger)
        {
            $this->logger = $logger;
        }

        /**
        * @param string $operation
        * @return void
        */
        public function consumerProcess(string $operation)
        {
            $this->logger->error($operation);
        }
    }
    ```
8.  Run `bin/magento setup:upgrade`
9.  Run `bin/magento setup:di:compile`
10. Send a message to queue
11. Run `bin/magento queue:consumer:list` and you must see your consumer at the end of the results
12. (optional) You can run `bin/magento queue:consumers:start <NAME_OF_CONSUMER>`

    