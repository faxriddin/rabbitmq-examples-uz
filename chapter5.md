#Мавзулар
##(Go RabbitMQ клиентини қўллаган ҳолда)

Аввалги қўлланмада биз бизнинг қайдлаш тизимимизни мукаммаллаштиргандик. Фақат бошланғич содда трансляцияни амалга ошира оладиган fanout exchange ўрнига биз direct exchange ни қўллаб уни қайдларни саралаб қабул қилишгача қайта ишлагандик.

Биз direct exchange  ёрдамида уни мукаммаллаштирган бўлсакда, у ҳам камчиликга эга бўлиб,  у бир қанча шартлар асосида маршрутлашни амалга ошира олмайди.

Бизнинг қайдлаш тизимимизда биз обуна бўлиш учун нафақат муҳимликка (severity), балки яна қайдларни тарқатувчи source (манба) га асосланишимиз ҳам мумкин. Сиз эҳтимол биларсиз ушбу концепция маршрутлашни муҳимлик (severity (info/warn/crit...)) ва объектга (facility (auth/cron/kern...)) асосланувчи unix ускунаси бўлмиш syslog дан олинган. 

Бу бизга анча мослашувчанликни беради. Биз 'cron' дан келаётган критик хатоларнигина ва яна 'kern' дан келаётган барча ҳабарларни эшитишни хоҳлашимиз мумкин.

Буни бизнинг қайдлаш тизимимизда амалга ошириш учун биз янада мукаммаллашган topic exchange (мавзули айирбошлаш) ни ўрганишимиз керак. 

#Мавзули айирбошлаш

Topic exchange га юборилган ҳабарлар routing_key лари ихтиёрий кўринишда эмас, балки бир-биридан нуқталар билан ажратилган сўзлардан иборат бўлиши керак. Сўзлар ихтиёрий бўлиши мумкин, бироқ одатда улар ҳабарларга боғлиқ бўлган баъзи муҳим нарсаларни белгилаб беради. Уларга мисол тариқасида қуйидагиларни олиш мумкин: "stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit". Унда 255 байтгача сиз хоҳлаган кўринишдаги routing key бўлиши мумкин.

Binding key ҳам шу шаклда бўлиши керак. Topic exchange мантиғи direct exchange никига ўхшаш бўлиб, ҳабар аниқ routing key билан жўнатилиб, binding key га мос келадиган навбатларга тақсимланади. Бироқ binding key лар учун иккита хусусий ҳолат бор:

•	* (star(юлдузча)) фақат битта сўзни алмаштира олади.
•	# (hash(панжара)) 0 ёки ундан кўп сўзларни алмаштира олади.

Буни мисолларда тушунтириш онсонроқ:

![](5.1.png)

Ушбу мисолда биз барча ҳайвонларни таърифловчи ҳабарларни жўнатмоқчимиз. Ҳабарлар учта сўзни (иккита нуқтали)ўзида мужассамлаштирган routing key билан жўнатилади. Routing key даги биринчи сўз speed (тезликни), иккинчиси colour (рангни) ва учинчиси species(турларни) англатади: `<speed>.<colour>.<species>`.

Биз учта боғланишни яратдик: Q1 "*.orange.*" binding key билан боғланган, Q2 эса "*.*.rabbit" ва "lazy.#" лар билан боғланган.

Ушбу боғланишларни қуйидагича умумлаштириш мумкин:

•	Q1 барча orange ҳайвонларга қизиққан.
•	Q2 эса rabbits (қуёнлар) ва lazy (эринчоқ) ҳайвонлар ҳақидаги барча ҳабарларни эшитади.

"quick.orange.rabbit" қийматли routing key лар иккала навбатларга ҳам тақсимланади. "lazy.orange.elephant" ҳабар ҳам иккаласига боради. Бошқа томондан "quick.orange.fox" фақат биринчи навбатга "lazy.brown.fox" эса фақат иккинчи навбатга тақсимланади. "lazy.pink.rabbit" иккинчи навбатга бир марта етказилади гарчи у иккита боғланишга эга бўлса ҳам. "quick.brown.fox" ҳеч қайси боғланишга мос келмаганлиги боис у ташлаб юборилади.

Нима содир бўлади агарда биз контрактни бузсак ва ҳабарни битта ёки тўртта сўз билан яъни, "orange" ёки "quick.orange.male.rabbit" каби жўнатсак? Бу ҳабарлар ҳеч қайси боғланишга мос келмаганлиги боис ташлаб юборилади.

Бошқа томондан "lazy.orange.male.rabbit" у тўртта сўзли бўлса ҳам охирги боғланишга мос келади ва иккинчи навбатга тақсимланади.

##Мавзули айирбошлаш (Topic exchange)

Topic exchange тўлақонли ҳисобланади ва у ўзини бошқа exchange лар каби тута олади.

Агар навбат "#" (панжарали) ли binding key га боғланган бўлса у ҳолда у routing key ихтиёрий кўринишда бўлган тақдирда ҳам у ҳабарларни қабул қилади ва у routing key га боғлиқсиз равишда - fanout exchange га ўхшайди.

Агар боғланишлар "*" (юлдузча) ва "#" (панжара) махсус белгиларни қўлламаса, topic exchange ўзини direct exchange каби тута бошлайди.

#Барчасини бирга қўйсак

Биз topic exchange ни қайдлаш тизимимизда қўлламоқчи бўлаябмиз. Биз ишчи ҳолат - routing key лар иккита сўзни ўзида мужассамлаштирган ҳолатдан бошлаймиз: `<facility>.<severity>`.

Код аввалги код га ўхшаш.

emit_log_topic.go коди:
```
package main

import (
        "fmt"
        "log"
        "os"
        "strings"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_topic",          // exchange
                severityFrom(os.Args), // routing key
                false, // mandatory
                false, // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 3) || os.Args[2] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[2:], " ")
        }
        return s
}

func severityFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "anonymous.info"
        } else {
                s = os.Args[1]
        }
        return s
}
```
receive_logs_topic.go коди:

```
package main

import (
        "fmt"
        "log"
        "os"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        err = ch.ExchangeDeclare(
                "logs_topic", // name
                "topic",      // type
                true,         // durable
                false,        // auto-deleted
                false,        // internal
                false,        // no-wait
                nil,          // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // no-wait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

        if len(os.Args) < 2 {
                log.Printf("Usage: %s [binding_key]...", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_topic", s)
                err = ch.QueueBind(
                        q.Name,       // queue name
                        s,            // routing key
                        "logs_topic", // exchange
                        false,
                        nil)
                failOnError(err, "Failed to bind a queue")
        }

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto ack
                false,  // exclusive
                false,  // no local
                false,  // no wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        log.Printf(" [x] %s", d.Body)
                }
        }()

        log.Printf(" [*] Waiting for logs. To exit press CTRL+C")
        <-forever
}
```
Барча қайдларни қабул қилиш:

```
$ go run receive_logs_topic.go "#"
```
"kern" объектидан барча қайдларни қабул қилиш:

```
$ go run receive_logs_topic.go "kern.*"
```
Ёки сиз агар фақат барча "critical" қайдларни эшитиш учун:


