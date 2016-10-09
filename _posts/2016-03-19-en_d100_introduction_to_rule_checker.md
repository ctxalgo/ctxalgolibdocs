---
title: Introduction to rule checker
layout: post
category: docs_rule_checker
language: en
---

It is often necessary to apply certain rules on trading strategy executions. For example, one rule from the exchange
may require that the number of order requests per account per trading day must not exceed a certain quota. To enforce
this requirement, every order from strategies operating on the same account are sent to a rule checker before they
are handed over to the order execution component. If the order request does not satisfy this rule, it is rejected.

This article introduces the rule checker in the CTX platform.
The following figure shows the architecture of the rule checker. Nodes in the figure represent processes. Different processes are marked with a different border color. For example, `rule_checker` runs in a process, while `input_proxy` and `output_proxy` runs in another process. Edges represent zeromq queues.

![rule checker architecture](/rule_checker/images/rule_checker_architecture.png "Rule checker architecture")

## The input and output proxy

The input and output proxy allows all messages producers (those messages will be sent to rule checkers) and rule checkers result consumers to have a single queue to connect to. This is the [Dynamic Discovery Problem](http://zguide.zeromq.org/page:all#The-Dynamic-Discovery-Problem) mentioned in the zeromq documentation.

`input_proxy` and `output_proxy` runs in the same process, they opens up four zeromq queues:

1. The input queue to the `input_proxy`. This queue accepts messages from various sources, such as strategies, market brokers. The input proxy allows different messages sources to write into the same queue, meaning that different sources only need to know a single queue to send messages to.

2. The output queue from the `input_proxy`. This queue will output all messages from the input queue. Multiple rule checker processors can subscribe this queue to get messages.

3. The input queue to the `output_proxy`. Multiple rule checkers outputs rule checking results into this input queue. The `output_proxy` allows multiple rule checkers to know a single queue to send results to.

4. The output queue from the `output_proxy`. This queue passes all rule checking result messages from all the connected rule checkers. Different rule checking result consumers, such as the altera visualizer and database storage subscribe to this queue.

To start the input and output proxy, run the script in

```
ctxalgolib.scripts.rule_checking.start_rule_checker_proxies.py
```

In a console, type

```
python start_rule_checker_proxies.py
```

to see its command line options:

```
Options:
  -h, --help            show this help message and exit
  --address=ADDRESS     string, the address of the proxies. Default:
                        tcp://127.0.0.1
  --input-proxy-sub-port=INPUT_PROXY_SUB_PORT
                        int, the port for the input proxy subscriber port.
                        This is the port messages sources, such as strategies
                        and market brokers. Together with the --address
                        option, they form the full address:input-proxy-sub-
                        port.
  --input-proxy-pub-port=INPUT_PROXY_PUB_PORT
                        int, the port for the input proxy publisher port. This
                        is the port from which rule checker processors
                        subscribe messages. Together with the --address
                        option, they form the full address:input-proxy-pub-
                        port.
  --output-proxy-sub-port=OUTPUT_PROXY_SUB_PORT
                        int, the port for the output proxy subscriber port.
                        This is the port to which rule checker processes send
                        out checking result messages. Together with the
                        --address option, they form the full address:output-
                        proxy-sub-port.
  --output-proxy-pub-port=OUTPUT_PROXY_PUB_PORT
                        int, the port for the output proxy publisher port.
                        This is the port from which rule checker result
                        consumers subscribe rule checking results. Together
                        with the --address optoin, the form the full address
                        :output-proxy-pub-port.
```

The `--address` option defines the IP address of the rule checker proxies, and the four options for pub, sub ports define the four ports mentioned above.



## The rule checker process

The `rule_checker` process takes a set of rule configuration, for example, which rules to check and parameters to each rule, and check those rules against source messages from the `input_proxy`. If there is a rule violation, the rule checker will send those violations to the `output_proxy`.

You can start multiple rule checker processors and they all connect to the same input and output proxies. Different rule checkers may run in different machines and configured with different set of rules. For example, if there are many rules to check and a single CPU core is not able to handle the computation, then you can start multiple rule checker processors, and config each rule checker with part of the rules.

The rule checker is configured with a pass-through-queue, which allows some raw messages to pass-through the rule checker and arrive, for example, to the `order_executor` process. For example, if an order request does not violate any rule, then it is passed to the `order_executor` so the order can be executed.

To start the rule checker, run the script in

```
ctxalgolib.scripts.rule_checking.check_rules.py
```

Please refer to the command line options for this script to see how to start it properly.

In a console, type
```
python check_rules --help
```

to see its command line options:

```
Usage: check_rules.py [options]

Options:
  -h, --help            show this help message and exit
  --rules=RULES         Path to a rule configuration file. The configuration
                        file should contain a JSON dict object of type
                        dict{string: dict{string: object}}, meaning
                        dict{rule_name: dict{parameter_name:
                        parameter_value}}.
  --exit-time=EXIT_TIME
                        Rule checker exit time in form of HH:MM:SS or yyyy-mm-
                        ddTHH:MM:SS. If in form of HH:MM:SS, it means the rule
                        checker should exit at that time of the current
                        trading day. When in form of yyyy-mm-ddTHH:MM:SS, it
                        means the rule checker will exit at that date and
                        time. Default: 15:30:00.
  --input-queue=INPUT_QUEUE
                        The input queue to the rule checker, which connects to
                        the output of the input proxy. Default:
                        tcp://127.0.0.1:5560
  --output-queue=OUTPUT_QUEUE
                        The output queue from the rule checker, which connects
                        to the input of the output proxy. Default:
                        tcp://127.0.0.1:6555
  --raw-message-queue=RAW_MESSAGE_QUEUE
                        The queue to which raw messages are sent. Raw messages
                        such as strategy orders.Default: tcp://127.0.0.1:5555
  --debug               If present, print debugging information. Default:
                        False
  --status-report-frequency=STATUS_REPORT_FREQUENCY
                        The number of seconds between each status report
                        request. At this frequency, a status report message is
                        sent to rules that listen to such messages. A rule
                        will respond to this message by sending out a status
                        report. Default: 60
  --passing-through-queue=PASSING_THROUGH_QUEUE
                        The passing-through queue used to let order requests
                        to reach the order execution layer. If not present, no
                        raw message will be passed through. Default: None,
                        meaning that no passing-through-queue is provided.
  --passing-through-topics=PASSING_THROUGH_TOPICS
                        Comma-separated topics for messages that should be
                        passed through to the passing-through-queue in case
                        there is no rule violation. This option only has
                        effect when the --passing-through-queue option is
                        specified. Note, the topics here will be matched using
                        exact string matching, this is different from the
                        zeromq string prefix matching semantics for message
                        filters. Default: STRATEGY.ORDER
```

The `--rules` option specifies a file containing rule configurations. The configuration is a JSON dict object. As an example, the following JSON object:

```
{
    "max_order_quota": {"default_quota": 10}
}
```

specifies a single rule named `max_order_quota` which has a parameter `default_quota` with value 10.


## Other processors in the figure

The other processors in the figure provide a full picture of how the whole system are connected.

The `strategy` process is an algo-trading strategy, it sends source messages such as order requests to the rule checker, and the rule checker can apply flow-control rules, for example, the order request speed cannot exceed 1000 order/minute.

The `market_broker` process retrieves full market tick data and sends those tick data to the rule checker, to enable rules such as the following: a limit order request cannot request a price which is some percentage away from the latest known price.

The `order_executor` process will execute the order requests which do not violate any rule.

The `alerta_visualizer` process visualizes the rule checker results using [alerta.io](http://alerta.io/).

The `database` process storage the rule checker results into database.

## Running the rule checker

The correct order to start the various processors is to follow the reversed edges of the figure:

1. Start the `alterta_visualizer`, `database` processors and the `order_executor` processes.

2. Start the input and output proxies (these two runs in the same process).

3. Start the `rule_checker` process.

4. Start the `market_broker` and `strategy` processes.
