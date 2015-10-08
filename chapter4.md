#Маршрутлаш
##(Go RabbitMQ клиентини қўллаган ҳолда)
Аввалги қўлланмада биз соддагина қайдловчи тизимни қурдик. Биз қайд ҳабарларини бир қанча қабул қилувчиларга трансляция қилишга қодир эдик.

Ушбу қўлланмада биз унга қўшимча имконият қўшишга яъни биз уни фақат қисм тўпламдаги ҳабарларга ёзилишга имкон берадиган қилишга отланаябмиз. Масалан, биз фақат критик хатоларни қайд файлига (диск сақлаш жойига) йўналтиришимиз мумкин бўлади. Хозирда ҳамма қайд ҳабарлар консолга чоп этилаябди.

#Боғланишлар (bindings)
Аввалги мисолда биз боғланишларни яратгандик.  Сиз кодни ёдга солишингиз мумкин:

```
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil)
```
binding бу exchange ва queue орасидаги ўзаро боғловчи. Бу қуйидагича ўқилиши ҳам мумкин: ушбу exchange даги ҳабарларга навбат қизиқаябди.

Binding лар қўшимча routing_key параметрларни қабул қилиши мумкин. Channel.Publish  параметри билан чалкашиш бўлмаслиги сабабли биз уни binding key деб номлаймиз. Қуйида биз боғланишни калит билан қандай яратишимиз кўрсатилган:

```
err = ch.QueueBind(
  q.Name,    // queue name
  "black",   // routing key
  "logs",    // exchange
  false,
  nil)
```
binding key маъноси exchange турига боғлиқ. fanout exchange да ушбу қиймат ташлаб кетилади.

#Exchange ни йўналтириш
Бизнинг аввалги қўлланмада яратган қайдлаш тизимимиз ҳамма ҳабарларни ҳамма қабул қилувчиларга трансляция қилади. Биз уни кенгайтирмоқчимиз, яъни ҳабарларни уларнинг муҳимлигига қараб саралаш имконини қўшмоқчимиз. Масалан, биз қайд ҳабарларидан фақат критик хатолиларини дискка ёзадиган қилишимиз ва ортиқча эслатмали ҳабарлар билан дискни тўлдирмайдиган қилишимиз мумкин.

Биз қўллаган fanout exchange кўп ҳам мослашувчан эмас бўлиб, у фақат ўйламасдан трансляция қилишгагина қодир.

Биз уни ўрнига direct exchange ни қўллаймиз. direct exchange  алгоритми жуда содда бўлиб, у ҳабарларни уларнинг routing key ига  навбатларнинг binding key (боғланиш калити) аниқ мос келсагина маршрутлайди.

Буни иллюстрация қилиш учун навбатдаги созлашни кўриб чиқамиз:

![](4.1.png)

Бу созлашда кўриб турганингиздек биз X  direct exchange иккита навбат билан боғланган. Биринчи навбат orange  номли binding key билан боғланган. Иккинчи навбат эса иккита боғланиш билан боғланган: биринчисининг binding key номи black ва иккинчисиники эса green. 

Бу созлашда ҳабар exchange га orange номли routing key билан нашр қилинган бўлиб, у Q1 навбатга маршрутланади. routing key номи black ёки green бўлганларр Q2 навбатга маршрутланади. Барча бошқа ҳабарлар бекор қиланиди.

#Кўп боғлашлар

![](4.2.png)

Бу ҳақиқатдан ҳам мумкин. Яъни кўп навбатларни бир хил номли binding key билан боғлаш мумкин. Бизнинг мисолда биз X ва Q1 орасида black номли binding key ни қўлладик. Ушбу ҳолатда direct exchange ҳудди fanout каби ишлайди ва ҳабарларни мос келувчи навбатларга жўнатади. routing key калити black бўлган ҳабарлар Q1 ва Q2 навбатларга тақсимланади.

#Қайдларни тарқатиш

Биз бу моделни қайдлаш тизимимизда ишлатамиз. fanout ўрнига биз энди ҳабарларни direct exchange га жўнатамиз. Биз қайд муҳимлигини routing key да жўнатамиз. Шундай йўл билан биз қабул қилиш учун хоҳлаган сценарийни яратиб оламиз. Келинг биринчи тарқатилаётган ҳабарларга қараймиз.

Ҳар доимгидек биз аввал exchange яратишимиз керак:

```
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
```
Ва биз ҳабарни жўнатишга тайёрмиз:

```
err = ch.ExchangeDeclare(
  "logs_direct", // name
  "direct",      // type
  true,          // durable
  false,         // auto-deleted
  false,         // internal
  false,         // no-wait
  nil,           // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs_direct",         // exchange
  severityFrom(os.Args), // routing key
  false, // mandatory
  false, // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
  })
```
Соддалик учун биз 'severity'(муҳимлик) ни 'info', 'warning' ва 'error' лардан бири каби ифодалашимиз мумкин деб оламиз.

#Обуна бўлиш (Subscribing)

Ҳабарларни қабул қилиш аввалгидек ишлайверади фақат битта фарқи шундаки, биз ҳар бир бизни қизиқтирган муҳимлик учун алоҳида боғланишларни яратамиз.
```
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
  log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
  os.Exit(0)
}
for _, s := range os.Args[1:] {
  log.Printf("Binding queue %s to exchange %s with routing key %s",
     q.Name, "logs_direct", s)
  err = ch.QueueBind(
    q.Name,        // queue name
    s,             // routing key
    "logs_direct", // exchange
    false,
    nil)
  failOnError(err, "Failed to bind a queue")
}
```
#Барчасини биргаликда қўйсак

![](4.3.png)

emit_log_direct.go сценарийсининг коди:
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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs_direct",         // exchange
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
                s = "info"
        } else {
                s = os.Args[1]
        }
        return s
}
```
receive_logs_direct.go коди:

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
                "logs_direct", // name
                "direct",      // type
                true,          // durable
                false,         // auto-deleted
                false,         // internal
                false,         // no-wait
                nil,           // arguments
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
                log.Printf("Usage: %s [info] [warning] [error]", os.Args[0])
                os.Exit(0)
        }
        for _, s := range os.Args[1:] {
                log.Printf("Binding queue %s to exchange %s with routing key %s",
                        q.Name, "logs_direct", s)
                err = ch.QueueBind(
                        q.Name,        // queue name
                        s,             // routing key
                        "logs_direct", // exchange
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
