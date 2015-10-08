#Узоқ масофадаги процедурани чақириш (Remote procedure call (RPC))
##(Go RabbitMQ клиентни қўллаган ҳолда)

Иккинчи қўлланмада биз Work Queues ни вақт-қамровида қийин масалаларни бир нечта ишчига тақсимлашни ўргангандик.

Лекин, биз узоқ масофадаги компьютерда функцияни чақирсакда ва натижани кутсак нима бўлади? Бу энди умуман бошқа йўналиш. Ушбу модель узоқ масофадаги процедурани чақириш (Remote Procedure Call ёки RPC) каби таниқли.

Ушбу қўлланмада биз RabbitMQ ни RPC тизимни яратишш учун қўлламоқчимиз: клиент ва масштаблашувчи RPC сервер. Биз айни дамда ҳеч қандай мураккаб масалага дуч келмаётганимиз учун биз фибоначчи сонларини қайтарадиган ясама RPC хизматини яратмоқчимиз.

#Навбатни қайта чақириш (Callback queue)

Умуман олганда RPC ни RabbitMQ устида яратиш онсон. Клиент сўров билан ҳабар жўнатади ва сервер ҳабар билан жавоб беради. Жавобни олиш учун биз 'callback' навбат манзилини сўров билан бирга жўнатамиз. Биз одатдаги навбатни қўллашимиз мумкин. Келинг буни қилишга уриниб кўрамиз:
```
q, err := ch.QueueDeclare(
  "",    // name
  false, // durable
  false, // delete when usused
  true,  // exclusive
  false, // noWait
  nil,   // arguments
)

err = ch.Publish(
  "",          // exchange
  "rpc_queue", // routing key
  false,       // mandatory
  false,       // immediate
  amqp.Publishing{
    ContentType:   "text/plain",
    CorrelationId: corrId,
    ReplyTo:       q.Name,
    Body:          []byte(strconv.Itoa(n)),
  })
```
##Ҳабар ҳислатлари
AMQP протоколи олдиндан белгилаб қўйилган 14 та ҳислатни ҳабарга қўшиб жўната олади. Қуйидагилардан бошқа кўплари кам ишлатилади:

•	persistent(турғунлик): Ҳабарни турғун ҳолатда (true қийматда) ёки алмашинувчан (false қийматда) белгилайди. Сиз эҳтимол буни иккинчи қўлланмадан билсангиз керак.

•	content_type: mime-type кодлаштиришни аниқлайди. Масалан, кодлаш учун амалда кўп синалган JSON ишлатилади ва уни қўллаш учун application/json дан фойдаланинг.

•	reply_to: callback навбат номини кўрсатиш учун қўлланилади.

•	correlation_id: сўровларни RPC жавоблари билан мослаштириш учун фойдали ҳисобланади.

##Ўзаро боғланиш ID (Correlation Id)

Юқорида келтирилган методда биз ҳар бир RPC сўров учун callback навбатни яратамиз. Бу етарли даражада самара беради, лекин унданда яхши йўл бор – келинг ҳар бир клиент учун битта callback навбат яратамиз.

Бу янги муаммони кўтаради, жавобни қабул қилиб қайси сўровга бу жавоб тегишли эканлигини аниқлаб бўлмайди. Мана қачон correlation_id ишлатилади. Биз унга ҳар бир сўров учун такрорланмас қиймат бермоқчимиз. Кейинроқ, қачонки биз callback навбатда ҳабар қабул қилганимизда биз ушбу ҳислатга эътибор беоамиз ва унга қараб қайси жавоб қайси сўровга тегишли эканлигини аниқлаштириб оламиз. Агар биз номаълум correlation_id  қийматни кўрсак биз балки хавфсизлик билан ушбу сўровга боғлиқ бўлмаган ҳабарни бекор қилишимиз мумкин.

Сиз савол беришингиз мумкин: нимага биз уларни callback навбатда аниқланмаган ҳабар дея йўқ қилишимз керак ва нима учун етишмовчилик хатолиги деб олмаябмиз уларни? Бу сервер томонда “пойга” имконияти ҳолатига боғлиқ. Аммо эҳтимол RPC сервер жавобни бизга жўнатгандан кейин ўлар, лекин у аввал сўров учун тасдиқ ҳабарини жўнатади. Агар бу юз берса у ҳолда қайта юкланган RPC сервер сўров жараёнини қайта ишлай бошлайди. Мана нима учун клиент томонда биз дубликатоар билан ундай муносабатда бўлишимиз керак. RPC яна идеал ҳолатда идемпотент* бўлиши керак.

*идемпотент – математик объектнинг ёки амалнинг хусусиятини англатувчи термин бўлиб, амални объектга қайта қўллаш яна бир хил натижа беришидир. 

#Хулоса

![](6.1.png)

Бизнинг RPC қуйидагига ўхшаб ишлайди:

•	Клиент ишга тушган вақтда у махсус callback навбат яратади. 

•	RPC  сўрови учун клиент ҳабарни иккита хоссаси билан жўнатади: callback навбатни қийматлаб олган reply_to ва ҳар бир сўров учун такрорланмас бўлган correlation_id.

•	Сўров rpc_queue навбатга жўнатилади.

•	RPC ишчи (яъни: сервер) ушбу навбатда сўровларни кутади. Агар сўров пайдо бўлса у бажариши керак бўлган ишини бажариб ҳабарни жавоб билан биргаликда reply_to майдонида кўрсатилган навбатга жўнатади.

•	Клиент callback навбатда маълумотларни кутади. Қачон ҳабар пайдо бўлса у correlation_id ни текширади. Агар у сўровдаги қиймат билан мос келса у ҳолда программага жавоб қайтарилади.

#Барчасини биргаликда қўйсак
Фибоначчи функцияси:

```
func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}
```
Биз фибоначчи функциясини эълон қилдик. У фақат бутун сондаги қийматни қабул қилади. (Буни катта сонга ишлашини кутманг ва яна эҳтимол бу энг секин ишлайдиган рекурсив функциядир).

Бизнинг RPC сервер кодимиз rpc_server.go қуйида келтирилган:

```
package main

import (
        "fmt"
        "log"
        "strconv"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
        }
}

func fib(n int) int {
        if n == 0 {
                return 0
        } else if n == 1 {
                return 1
        } else {
                return fib(n-1) + fib(n-2)
        }
}

func main() {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "rpc_queue", // name
                false,       // durable
                false,       // delete when usused
                false,       // exclusive
                false,       // no-wait
                nil,         // arguments
        )
        failOnError(err, "Failed to declare a queue")

        err = ch.Qos(
                1,     // prefetch count
                0,     // prefetch size
                false, // global
        )
        failOnError(err, "Failed to set QoS")

        msgs, err := ch.Consume(
                q.Name, // queue
                "",     // consumer
                false,  // auto-ack
                false,  // exclusive
                false,  // no-local
                false,  // no-wait
                nil,    // args
        )
        failOnError(err, "Failed to register a consumer")

        forever := make(chan bool)

        go func() {
                for d := range msgs {
                        n, err := strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")

                        log.Printf(" [.] fib(%d)", n)
                        response := fib(n)

                        err = ch.Publish(
                                "",        // exchange
                                d.ReplyTo, // routing key
                                false,     // mandatory
                                false,     // immediate
                                amqp.Publishing{
                                        ContentType:   "text/plain",
                                        CorrelationId: d.CorrelationId,
                                        Body:          []byte(strconv.Itoa(response)),
                                })
                        failOnError(err, "Failed to publish a message")

                        d.Ack(false)
                }
        }()

        log.Printf(" [*] Awaiting RPC requests")
        <-forever
}
```
Сервер томондаги код жуда содда:

•	Одатдагидек биз боғланиш, каналлар ва навбатни эълон қилишдан бошлаймиз.

•	Биз балким бирдан кўп сервер процессини ишга туширишни хоҳлармиз. Юкламаларни бир нечта серверга тақсимлаш учун каналда prefetch ни қийматлаш керак бўлади.

•	Биз навбатдан ҳабарларни қабул қилишимизда ишлатадиган каналларни олиш учун биз Channel.Consume ни ишлатаябмиз. Кейин биз goroutine га кириб ишларни бажариб жавобни ортга қайтариб жўнатамиз.

Бизнинг RPC клиент rpc_client.go кодимиз қуйидагича:

```
package main

import (
        "fmt"
        "log"
        "math/rand"
        "os"
        "strconv"
        "strings"
        "time"

        "github.com/streadway/amqp"
)

func failOnError(err error, msg string) {
        if err != nil {
                log.Fatalf("%s: %s", msg, err)
                panic(fmt.Sprintf("%s: %s", msg, err))
        }
}

func randomString(l int) string {
        bytes := make([]byte, l)
        for i := 0; i < l; i++ {
                bytes[i] = byte(randInt(65, 90))
        }
        return string(bytes)
}

func randInt(min int, max int) int {
        return min + rand.Intn(max-min)
}

func fibonacciRPC(n int) (res int, err error) {
        conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
        failOnError(err, "Failed to connect to RabbitMQ")
        defer conn.Close()

        ch, err := conn.Channel()
        failOnError(err, "Failed to open a channel")
        defer ch.Close()

        q, err := ch.QueueDeclare(
                "",    // name
                false, // durable
                false, // delete when usused
                true,  // exclusive
                false, // noWait
                nil,   // arguments
        )
        failOnError(err, "Failed to declare a queue")

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

        corrId := randomString(32)

        err = ch.Publish(
                "",          // exchange
                "rpc_queue", // routing key
                false,       // mandatory
                false,       // immediate
                amqp.Publishing{
                        ContentType:   "text/plain",
                        CorrelationId: corrId,
                        ReplyTo:       q.Name,
                        Body:          []byte(strconv.Itoa(n)),
                })
        failOnError(err, "Failed to publish a message")

        for d := range msgs {
                if corrId == d.CorrelationId {
                        res, err = strconv.Atoi(string(d.Body))
                        failOnError(err, "Failed to convert body to integer")
                        break
                }
        }

        return
}

func main() {
        rand.Seed(time.Now().UTC().UnixNano())

        n := bodyFrom(os.Args)

        log.Printf(" [x] Requesting fib(%d)", n)
        res, err := fibonacciRPC(n)
        failOnError(err, "Failed to handle RPC request")

        log.Printf(" [.] Got %d", res)
}

func bodyFrom(args []string) int {
        var s string
        if (len(args) < 2) || os.Args[1] == "" {
                s = "30"
        } else {
                s = strings.Join(args[1:], " ")
        }
        n, err := strconv.Atoi(s)
        failOnError(err, "Failed to convert arg to integer")
        return n
}
```
rpc_client.go ва rpc_server.go  кодларимизни биргаликда тўлиқ кўришнинг айни вақти етиб келди.

Бизнинг RPC сервисимиз тайёр бўлди. Биз серверни ишга туширишимиз мумкин:

```
$ go run rpc_server.go
 [x] Awaiting RPC requests
```
Фибоначчи сонини олиш учун клиентни ишга туширинг:

```
$ go run rpc_client.go 30
 [x] Requesting fib(30)
```
Бу ерда келтирилган йўл нафақат RPC га хизмат кўрсатиш учун балким у бир қанча муҳим имкониятларга эга:

•	Агар RPC сервер жуда секин ишлаётган бўлса сиз масштаблаш мақсадида яна биттасини ишга туширишингиз мумкин. Иккинчи rpc_server.go ни терминалда ишга туширишга уриниб кўринг.

•	Клиент томонда RPC битта ҳабарни жўнатади ва қабул қилади. Натижада RPC клиент фақат битта тармоқ айланма вақтида битта сўров бажаради.
