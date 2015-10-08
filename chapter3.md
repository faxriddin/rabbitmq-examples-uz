#Publish (Нашр)/Subscribe (Обуна)
##(Go RabbitMQ клиентни қўллаган ҳолда)
Аввалги қўлланмада биз иш навбатни яратдик. Иш навбатининг пировардида ҳар бир масала аниқ битта ишчига етказилганди. Ушбу бу бўлимда биз умуман бошқача йўл тутамиз – биз ҳабарни бир қанча истъемолчиларга етказамиз. Ушбу модель "publish(нашр)/subscribe(обуна)" дея номланган.

Тасаввур хосил қилиш учун биз қайдлаш содда тизимини қурамиз. У иккита дастурдан ташкил топган бўлади – биринчиси ҳабарлар журналини нашр қилади иккинчиси уларни қабул қилиб чоп этади.

Бизнинг қайдлаш тизимимиздаги ҳар бир ишга тушган қабул қилувчи дастуримиз ҳабарларни қабул қилади. Шу тарзда биз битта қабул қилувчини ишга тушириб, қайдни дискка йўналтиришимиз ва бир вақтда биз бошқа қабул қилувчини ҳам ишга туширамиз ва қайдларни экранда кўрамиз.

Мазмунан, нашр қилинган қайд ҳабарлар ҳар бир қабул қилувчиларга трансляция қилинади.

#Алмашинувлар

Қўлланманинг аввалги қисмларида биз ҳабарларни навбатдан юбориб навбатдан қабул қилгандик. Rabbit да хабарлашишнинг тўлиқ модели билан танишиш вақти етиб келди.

Келинг аввалги қўлланмаларда нималар билан танишганимизни тез кўриб чиқайлик:

•	producer (юборувчи) бу – дастур фойдаланувчиси бўлиб, у ҳабарларни юборади.
•	queue (навбат) бу – хабарларни сақловчи буффер.
•	consumer (қабул қилувчи) бу – дастур фойдаланувчиси бўлиб, ҳабарларни қабул қилади.

RabbitMQ нинг ҳабарлашиш моделининг туб ўзак ғоясида – жўнатувчи ҳеч қачон ҳеч қанақа ҳабарларни навбатга жўнатмайди. Аслини олганда, жуда кўп ҳолларда юборувчи ҳабарларни қайси навбатга тақсимланганлигини билавермайди.

Буни ўрнига producer ҳабарларни exchange (алмашиниш)га юбориши мумкин. Exchange  бу – жуда содда нарса. У бир томондан producer лардан ҳабарни олса иккинчи томондан  уларни навбатга тиқади. Exchange олган ҳабарини аниқ нима қилиши кераклигини билиши керак. У аниқ бир навбатга қўшилиши керакми? У кўп навбатларга қўшилиши керакми? Ёки ушбу олинган ташлаб юбориладими. Булар учун қоидалар exchange type  орқали аниқланган.

![](3.1.png)

Бир қанча exchange турлари мавжуд: direct (узлуксиз), topic(мавзули), headers(сарлавҳалар) ва fanout(елпиғич каби тармоқланиш). Биз улардан биттаси охиргиси fanout га эътиборимизни қаратамиз. Келинг exchange ни ушбу турда яратамиз ва уни logs  деб номлаймиз:

```
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
```
fanout exchange бу – жуда оддий. Номидан билган бўлсангиз керак, у ўзи билган барча навбатларга тегишли ҳабарларни фақат трансляция қилади. Ва бу бизнинг қайдловчимиз учун айнан керак бўлган нарса.

#Exchange ларни рўйҳатлаш
Exchange ларни рўйҳатлаш учун доим фойдали бўлган rabbitmqctl ни ишга туширишингиз мумкин:

```
$ sudo rabbitmqctl list_exchanges
Listing exchanges ...
        direct
amq.direct      direct
amq.fanout      fanout
amq.headers     headers
amq.match       headers
amq.rabbitmq.log        topic
amq.rabbitmq.trace      topic
amq.topic       topic
logs    fanout
...done.
```
Ушбу рўйҳатда баъзи amq.* exchange лар ва (unnamed (номсиз)) одатдаги exchange лар бор. Улар одатий ҳолатда яратилади, лекин уларни сиз айни дамда ишлатишингиз амримаҳол.

#Номсиз exchange

Биз қўлланманинг аввалги қисмида exchange лар ҳақида ҳеч нарса билмасдик, аммо барибир навбатларга ҳабар жўната олдик. Бундай қила олганимизнинг боиси биз одатда яратиладиган ва бўш сатр (“”) билан номланадиган exchange ни ишлатдик.

Аввал биз ҳабарларни қандай қилиб нашр қилганлигимизни эсга олинг:

```
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
    ContentType: "text/plain",
    Body:        []byte(body),
  })
```
Бу ерда биз номсиз ёки одатдаги exchange ни қўллаябмиз: ҳабар навбатга агар мавжуд бўлса routing_key  параметридаги ном билан йўналтирилади.

Энди биз номланган exchange имизда нашр қилишимиз мумкин:

```
err = ch.ExchangeDeclare(
  "logs",   // name
  "fanout", // type
  true,     // durable
  false,    // auto-deleted
  false,    // internal
  false,    // no-wait
  nil,      // arguments
)
failOnError(err, "Failed to declare an exchange")

body := bodyFrom(os.Args)
err = ch.Publish(
  "logs", // exchange
  "",     // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing{
          ContentType: "text/plain",
          Body:        []byte(body),
  })
```
#Вақтинчаликли навбатлар
Сизни ёдингизда бўлса керак аввалги дарсларда биз номланган навбатларни қўллаган эдик (hello ва task_queue эслайсизми?). Ўша вақтда биз ишчига қайси навбат эканлигини кўрсатиш учун шундай қилишга мажбур эдик. Навбатга ном бериш юборувчи ва қабул қилувчи орасида навбатни эълон қилишда муҳимдир.

Аммо бу бизнинг қайдловчимизга тааллуқли эмас. Биз фақат қисм тўпламдагини эмас балки ҳамма хабарларни қайд қилиш ҳақида эшитишни хоҳлаймиз. Биз яна эски эмас балки айни дамда оқиб турган ҳабарларга қизиқаябмиз. Буни ҳал қилиш учун бизга икки нарса керак.

Биринчидан, қачон биз Rabbit га боғланар эканмиз бизга янги, бўш навбат керак. Бунинг учун биз тасодифий ном билан навбатни яратишимиз ёки янада яхшиси биз учун сервер номни танлаб навбатни яратиши мумкин. 

Иккинчидан агар биз қабул қилувчини узиб қўйсак навбат автоматик тарзда ўчиб кетиши керак.

Биз навбат номини бўш сатр билан қолдирган вақтимизда  биз номи генерция қилинган давомий бўлмаган навбатни amqp мижозда яратамиз:

```
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // no-wait
  nil,   // arguments
)
```
Метод қайтараётган вақтда навбат нусхаси RabbitMQ томонидан генерация қилинган тасодифий номга эга бўлади. Масалан, у қуйидаги кўринишда бўлиши мумкин: 

```
amq.gen-JzTY20BRgKO-HjmUJj0wLg.
```
Агар боғланиш ёпилиши ҳақида эълон берса, навбат ўчирилади. Чунки у эксклюзив сифатида эълон қилинган.

#Боғламлар

![](3.2.png)

Биз fanout exchange ва queue ларни яратдик. Энди биз exchange га бизнинг навбатга ҳабарларни юборишини айтишимиз керак. Бу exchange ва queue орасидаги муносабат binding (боғлам, боғланиш) деб номланади.

```
err = ch.QueueBind(
  q.Name, // queue name
  "",     // routing key
  "logs", // exchange
  false,
  nil
)
```
Хозирдан бошлаб logs exchange ҳабарларни queue га қўша бошлайди. 

##Боғламлар рўйҳати
Сиз боғламлар рўйҳатини яратишингиз мумкин, ҳа топдингиз бунинг учун rabbitmqctl list_bindings  дан фойдаланамиз.

#Барчасини биргаликда қўйсак

![](3.3.png)

log ҳабарларни жўнатувчи producer дастур аввалги қўлланма маълумотидан кўп фарқ қилмайди. Энг асосий фарқ шундаки, биз хозир ҳабарларимизни аввалгидек номсиз эмас балки logs номли exchange га нашр қилаябмиз. Биз жўнатишда routingKey ни қўйишимиз керак, аммо унинг қиймати fanout exchanges да эътиборга олинмайди. Қуйида emit_log.go сценарийсининг коди келитирилаябди:

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
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
        )
        failOnError(err, "Failed to declare an exchange")

        body := bodyFrom(os.Args)
        err = ch.Publish(
                "logs", // exchange
                "",     // routing key
                false,  // mandatory
                false,  // immediate
                amqp.Publishing{
                        ContentType: "text/plain",
                        Body:        []byte(body),
                })
        failOnError(err, "Failed to publish a message")

        log.Printf(" [x] Sent %s", body)
}

func bodyFrom(args []string) string {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "hello"
        } else {
                s = strings.Join(args[1:], " ")
        }
        return s
}
```
(emit_log.go манба)

Сиз кўриб турганингиздек, боғланишдан сўнг биз exchange ни эълон қилдик. Мавжд бўлмаган exchange га нашр қилиш мумкин бўлмаганлиги боис бу қадам зарур.

Агар exchange га навбат боғланмаган бўлса ҳабарлар ўчиб кетади ва бу ҳолат биз учун нормал ҳисобланади. Агар қабул қилувчи эшитмаётган бўлса биз ҳабарни хавфсизгина ўчиришимиз мумкин.

receive_logs.go учун код:

```
package main

import (
        "fmt"
        "log"

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
                "logs",   // name
                "fanout", // type
                true,     // durable
                false,    // auto-deleted
                false,    // internal
                false,    // no-wait
                nil,      // arguments
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

        err = ch.QueueBind(
                q.Name, // queue name
                "",     // routing key
                "logs", // exchange
                false,
                nil)
        failOnError(err, "Failed to bind a queue")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                true,   // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
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
(receive_logs.go манба)

Агар сиз қайдларни файлга сақламоқчи бўлсангиз, фақат console ни очинг ва қуйидагини ёзинг:

```
$ go run receive_logs.go > logs_from_rabbit.log
```
Агар сиз қайдларни экранда кўрмоқчи бўлсангиз у ҳолда янги терминал яратинг ва қуйидагини ишга туширинг:

```
$ go run receive_logs.go
```
Ва албатта, қайдларни жўнатиш учун қуйидагини ҳам ишга туширинг:

```
$ go run emit_log.go
```
rabbitmqctl list_bindings  ни қўллаб сиз кодингиз аниқ боғламларни яратганлигини ва навбатлар сиз хоҳлагандай эканлигига ишонч хосил қилишингиз мумкин. Ишга туширилган иккита receive_logs.go дастури ёрдамида сиз қуйидагига ўхшашни кўришингиз лозим:

