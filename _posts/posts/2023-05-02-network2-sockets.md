---
layout: post
title: "Сети ч.2. Linux network API aka Socket programming"
date: 2023-05-02 20:28:39 +0300
categories: [network, posts, sockets, linux]
---

Практическая сторона предыдущих частей о модели [OSI](network0-OSI) и [протоколах](network1-protocols)
выливается в написание своего клиент серверного приложения. 

# Кратко про Linux api
Linux глобально состоит из двух пространств: user и kernel space. Смысл заключается в том, что в ядре ОС имплементированы стеки протоколов
и доступа к их изменению у вас нет. Наружу разработчики ядра предоставили рычаги (API), дергая за которые, можно как-то взаимодействовать
с этими протоколами, чего хватает для написания пользовательских программ. Для работы с сетевым стеком существует достаточно много
низкоуровневых [системных вызовов](https://www.kernel.org/doc/html/v4.14/networking/kapi.html). В свою очередь каждый язык программирования
предлагает удобную обертку вокруг данных сетевых вызовов, чтобы код получался более надежным, читаемым и универсальным. Код,
который мы сегодня напишем, будет выглядеть схоже на всех языках программирования, так как мы будем работать близко к api операционной системы.
Мой любимый язык - Rust, я буду писать на нем, но я постараюсь объяснить каждую строчку так, чтобы не составляло труда переписать 
это на любом друге ЯП.

# Архитектура
1. Клиент должен уметь отправлять текстовое сообщение на сервер
2. Сервер должен уметь обрабатывать полученные сообщения
3. В случае, если сервер получит сообщение `ping`, он должен ответить клиенту `pong` и закрывать соединение

Схематично приложение будет выглядеть вот так:
![img.png](../../images/posts/network/Socket_Programming.png)
Здесь изображен стандартный стек системных вызовов клиент-серверного приложения, который можно найти по запросу Socket Programming.
Про сокеты я упомянул в [предыдущей части](network1-protocols). Еще раз повторюсь и скажу, что сокет - это абстракция.
В данном случае - это интерфейс взаимодействия со стеком TCP/IP, встроенным в ядро. Скажу еще, что приложение должно поддерживать
как работу с TCP, так и работу с UDP, чтобы посмотреть их возможности и различия на практике.

## Небольшая подготовка для гурманов
Для начала я напишу некоторую cli обертку, чтобы у нашей программы был удобный клиентский
интерфейс:
```rust
//main.rs

use clap::{Parser, ValueEnum};
use clap_num::number_range;

// Режимы, в которых будет работать приложение
#[derive(ValueEnum, Clone)]
enum Mode {
    Client,
    Server
}

// Выбор доступных протоколов
#[derive(ValueEnum, Clone)]
enum Proto {
    TCP,
    UDP
}

// Первые 1024 порта зарезервированы операционной системой, поэтому сразу напишем обработчик, валидирующий номер порта
fn port_validator(port: &str) -> Result<u16, String> {
    number_range(port, 1024, u16::MAX)
}


// Структура для аргументов командной строки
#[derive(Parser)]
struct Cli {
    #[arg(value_enum)]
    mode: Mode,
    #[arg(value_enum)]
    proto: Proto,
    #[clap(value_parser=port_validator)]
    port: u16
}

fn main() {
    let cli = Cli::parse();

    match cli.mode {
        Mode::Client => {
            println!("Client mode");
        }
        Mode::Server => {
            println!("Server mode");
        }
    }
}
```

```toml
#Cargo.toml
[package]
name = "client-server"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.3.21", features = ["derive"] }
clap-num = "1.0.2"
```

Собираем, запускаем:
```bash
cargo build

target/debug/client-server --help
Usage: client-server <MODE> <PROTO> <PORT>

Arguments:
  <MODE>   [possible values: client, server]
  <PROTO>  [possible values: tcp, udp]
  <PORT>   

Options:
  -h, --help  Print help

```

Запускаем уже в одном из режимов:
```bash
target/debug/client-server client tcp 65535
Client mode
```

Теперь переходим к написанию непосредственно серверной части
# Сервер
Основным входом в стек TCP/IP является системный вызов [socket](https://man7.org/linux/man-pages/man2/socket.2.html).
Данной функции необходимо сообщить о том, какое семейство протоколов мы хотим использовать (в нашем случае это IPv4) и
какой из протоколов нам интересен (TCP или UDP). Из документации видно, что количество возможностей и параметров гораздо обширнее, 
так как это базовый системный вызов. Далее из схемы выше видно, что идет вызов [bind](https://man7.org/linux/man-pages/man2/bind.2.html).
Он служит для привязки сокета к определенному адресу, состоящему из ip и port. Вот из-за этого вызова сокет часто и называют связкой
ip:port. Если бы мы писали на чистом Си, нам пришлось бы создавать некоторые структуры и вызывать данные функции буквально as is. 
В Rust это делается одной строчкой:

```rust
let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
```

Итого, функция запуска нашего сервера будет выглядеть таким образом:
```rust
fn start_server(proto: Proto, port: u16) {
    match proto {
        Proto::TCP => {
            let socket = TcpListener::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
        },
        Proto::UDP => {
            let socket = UdpSocket::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
        }
    }
}
```
## UDP
Дальше пути реализации серверов расходятся. Для UDP будет достаточно системного вызова [recvfrom](https://man7.org/linux/man-pages/man3/recvfrom.3p.html).
Данный вызов буквально считывает данные из открытого ранее сокета и не более того. Возвращает он размер прочитанного сообщения и адрес отправителя.
Так как у нас нет ни сессий ни потоков, и `recvfrom` срабатывает буквально один раз, то мы поместим его в бесконечный цикл, выход
из которого будет по определенному условию (ключевому сообщению). Также стоит отметить, что благодаря [внутренним событийным механизмам ядра](https://habr.com/ru/articles/416669/),
при отсутствии сообщения в сокете `recvfrom` просто встанет в режим ожидания и даже не будет тратить процессорное время. Итак, код:
```rust
fn start_server(proto: Proto, port: u16) {
    match proto {
        Proto::UDP => {
            let socket = UdpSocket::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
            // создаем буффер для входящего сообщения из 16-ти 8-ми битных ячеек.
            // Если длина сообщения превысит 16 байт, остальное просто отрежется
            let mut buf: [u8; 16] = [0; 16];
            // Стартуем бесконечный цикл, который закончится, если получим сообщение ping
            loop {
                // Читаем данные из сокета, если их нет, то recv_from будет ждать их поступления
                let (size, addr) = socket.recv_from(&mut buf).unwrap();
                // Конвертируем полученные байты в строку
                let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                println!("Got message from UDP client:\n{}", { message });
                socket.send_to("Server OK\n".as_bytes(), &addr).unwrap();
                // Если встречаем ключевое сообщение, отправляем в ответ pong и закрываем сокет
                if message == "ping"{
                    socket.send_to("pong".as_bytes(), &addr).unwrap();
                    break;
                }
            }
        }
    }
}
```

Как видно, udp - это очень прозрачно и просто. И уже можно проверить, то что это работает. Для этого запускаем наш сервер:
```bash
target/debug/client-server server udp 65535
```
Посмотреть, что сокет открылся и ждет сообщений можно с помощью утилиты:
```bash
lsof -iudp | grep 65535
client-se 23298 user    3u  IPv4 0xf0edabae6b4702a3      0t0  UDP localhost:65535
```
А с помощью утилиты `netcat` можно имитировать клиента:
```bash
nc -u 127.0.0.1 65535
```
После команды выше можно вводить любые сообщения и видеть их в консоли запуска сервера. Если отправить 
"ping", сервер ответит "pong" и выключиться.

## TCP
Если посмотреть схему выше, то для TCP сервера после создания сокета и привязки его к порту идет вызов [listen](https://man7.org/linux/man-pages/man2/listen.2.html).
Перевод из официальной документации следующий - помечает сокет как пассивный, принимающий запросы на соединение для вызова [accept](https://man7.org/linux/man-pages/man2/accept.2.html).
В этом и есть смысл данного протокола - тот сокет, что мы создали, всего лишь некий роутер для входящий соединений. Далее под каждого клиента
(сессию) создается свой сокет (в отличие от UDP). Посредством такого механизма и достигается stateful архитектура протокола.
Именно `accept` создает эти "сессионные" сокеты, обрабатывая очередь запросов на подключение. Но в Rust это конечно убрано
под капот и сделано проще и понятнее для разработчиков:

```rust
fn start_server(proto: Proto, port: u16) {
    match proto {
        Proto::TCP => {
            let socket = TcpListener::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
            /*
            Строкой выше вернулся уже "слушающий" новые соединения сокет (на котором был вызван стандартный listen()).
            Функция incoming() возвращает итератор для прохождения по коллекции полученных соединений.
            Под капотом вызывается стандартный accept() в блокирующем режиме. То есть, если соединений нет,
            accept() ожидает и процесс спит. Как только соединение пришло, оно добавляется в коллекцию и
            итератор тут же возвращает next(). Исходник:
            https://github.com/rust-lang/rust/blob/ce01f4d2e04f785018acd881161e58394ed3d522/library/std/src/net/tcp.rs#L1040
             */
            for stream in socket.incoming() {
                /*
                Еще одно отличие от UDP, что размер сообщения нигде не задается, то есть общение происходит непрерывно.
                Если в случае с UDP, recvfrom считывает ровно одну датаграмму, то здесь открывается
                поток. Поток означает заранее неизвестный размер сообщений, если размер превысит capacity буфера, то данные
                будут обработаны кусками, но потерь не произойдет. Точно также и в случае отправки - при отправке большого
                сообщения, TCP сегментирует его и сообщение дойдет по частям, а на другой стороне TCP склеит данные части
                в одно единое исходное сообщение и для пользователя это будет выглядеть, как если бы никакой сегментации не было.
                 */
                let mut stream = stream.unwrap();
                println!("New connection from: {}", stream.peer_addr().unwrap());
                loop {
                    let mut buf: [u8; 16] = [0; 16];
                    /*
                    Здесь происходит считывание строк из сокета, поэтому это также блокирующая
                    часть, пока соединение живое. Как только в сокете появились данные, срабатывает сигнал и
                    stream.read() считывает эти данные в буфер и разблокирует поток.
                     */
                    let size = stream.read(&mut buf).unwrap();
                    let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                    println!("Got a message from TCP client:\n{}", { message.clone() });
                    stream.write_all("Server OK\n".as_bytes()).unwrap();
                    if message == "ping" {
                        stream.write_all("pong".as_bytes()).unwrap();
                        return;
                    }
                }
            }
        },
        Proto::UDP => {
            let socket = UdpSocket::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
            // создаем буффер для входящего сообщения из 16-ти 8-ми битных ячеек.
            // Если длина сообщения превысит 16 байт, остальное просто отрежется
            let mut buf: [u8; 16] = [0; 16];
            // Стартуем бесконечный цикл, который закончится, если получим сообщение ping
            loop {
                // Читаем данные из сокета, если их нет, то recv_from будет ждать их поступления
                let (size, addr) = socket.recv_from(&mut buf).unwrap();
                // Конвертируем полученные байты в строку
                let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                println!("Got message from UDP client:\n{}", { message });
                socket.send_to("Server OK\n".as_bytes(), &addr).unwrap();
                // Если встречаем ключевое сообщение, отправляем в ответ pong и закрываем сокет
                if message == "ping"{
                    socket.send_to("pong".as_bytes(), &addr).unwrap();
                    break;
                }
            }
        }
    }
}
```

Серверная часть приложения полностью готова, его также можно потестить с помощью netcat, например:
```bash
nc localhost 65535
```
или 
```bash
echo "Client Hello!" | nc localhost 65535
```

Обратите внимание, что сервер является однопоточным, синхронным, блокирующимся. То есть до тех пор, пока активно одно соединение,
второе находится в очереди и не будет обработано.
## Client
Для подключения к TCP серверу клиент делает единственный вызов [connect](https://man7.org/linux/man-pages/man2/connect.2.html).
По сути это просто открытие сокета и привязка его к удаленному адресу и порту сервера. Затем через этот сокет и происходит все общение.
В случае с UDP это заменяется на элементарный вызов `send_to`. Однако стоит отметить, что в отличие от TCP, где сервер сам открывает дополнительные
сокеты, UDP клиенту придется задать порт самостоятельно, дабы избежать конфликта из-за тестирования на одной машине (UDP сервер займет сокет первым).
Привожу полный код приложения вместе с клиентский частью:

```rust
use std::io;
use std::io::{BufRead, Read, Write};
use std::net::{TcpListener, TcpStream, UdpSocket};
use clap::{Parser, ValueEnum};
use clap_num::number_range;

// Режимы, в которых будет работать приложение
#[derive(ValueEnum, Clone)]
enum Mode {
    Client,
    Server
}

// Выбор доступных протоколов
#[derive(ValueEnum, Clone)]
enum Proto {
    TCP,
    UDP
}

// Первые 1024 порта зарезервированы операционной системой, поэтому сразу напишем обработчик, валидирующий номер порта
fn port_validator(port: &str) -> Result<u16, String> {
    number_range(port, 1024, u16::MAX)
}


// Структура для аргументов командной строки
#[derive(Parser)]
struct Cli {
    #[arg(value_enum)]
    mode: Mode,
    #[arg(value_enum)]
    proto: Proto,
    #[clap(value_parser=port_validator)]
    port: u16
}


fn start_server(proto: Proto, port: u16) {
    match proto {
        Proto::TCP => {
            let socket = TcpListener::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
                /*
                Строкой выше вернулся уже "слушающий" новые соединения сокет (на котором был вызван стандартный listen()).
                Функция incoming() возвращает итератор для прохождения по коллекции полученных соединений.
                Под капотом вызывается стандартный accept() в блокирующем режиме. То есть, если соединений нет,
                accept() ожидает и процесс спит. Как только соединение пришло, оно добавляется в коллекцию и
                итератор тут же возвращает next(). Исходник:
                https://github.com/rust-lang/rust/blob/ce01f4d2e04f785018acd881161e58394ed3d522/library/std/src/net/tcp.rs#L1040
                 */
                for stream in socket.incoming() { 
                /*
                Еще одно отличие от UDP, что размер сообщения нигде не задается, то есть общение происходит непрерывно.
                Если в случае с UDP, recvfrom считывает ровно одну датаграмму, то здесь открывается
                поток. Поток означает заранее неизвестный размер сообщений, если размер превысит capacity буфера, то данные
                будут обработаны кусками, но потерь не произойдет. Точно также и в случае отправки - при отправке большого
                сообщения, TCP сегментирует его и сообщение дойдет по частям, а на другой стороне TCP склеит данные части
                в одно единое исходное сообщение и для пользователя это будет выглядеть, как если бы никакой сегментации не было.
                 */
                    let mut stream = stream.unwrap();
                    println!("New connection from: {}", stream.peer_addr().unwrap());
                    loop {
                        let mut buf: [u8; 16] = [0; 16];
                        /*
                        Здесь происходит считывание строк из сокета, поэтому это также блокирующая
                        часть, пока соединение живое. Как только в сокете появились данные, срабатывает сигнал и
                        stream.read() считывает эти данные в буфер и разблокирует поток.
                         */
                        let size = stream.read(&mut buf).unwrap();
                        let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                        println!("Got a message from TCP client:\n{}", { message.clone() });
                        stream.write_all("Server OK\n".as_bytes()).unwrap();
                        if message == "ping" {
                            stream.write_all("pong".as_bytes()).unwrap();
                            return;
                        }
                    }
                }
            },
        Proto::UDP => {
            let socket = UdpSocket::bind(format!("127.0.0.1:{port}", port=port)).unwrap();
            // создаем буффер для входящего сообщения из 16-ти 8-ми битных ячеек.
            // Если длина сообщения превысит 16 байт, остальное просто отрежется
            let mut buf: [u8; 16] = [0; 16];
            // Стартуем бесконечный цикл, который закончится, если получим сообщение ping
            loop {
                // Читаем данные из сокета, если их нет, то recv_from будет ждать их поступления
                let (size, addr) = socket.recv_from(&mut buf).unwrap();
                // Конвертируем полученные байты в строку
                let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                println!("Got message from UDP client:\n{}", { message });
                socket.send_to("Server OK\n".as_bytes(), &addr).unwrap();
                // Если встречаем ключевое сообщение, отправляем в ответ pong и закрываем сокет
                if message == "ping"{
                    socket.send_to("pong".as_bytes(), &addr).unwrap();
                    break;
                }
            }
        }
    }
}

fn start_client(proto: Proto, port: u16) {
    match proto {
        Proto::TCP => {
            // Делаем системный вызов connect, тем самым открывая поток на запись в сокет
            let mut stream = TcpStream::connect(format!("127.0.0.1:{port}", port=port)).unwrap();
            let mut buffer = String::new();
            let stdin = io::stdin();
            // Считываем строчки со стандартного ввода (терминала)
            while let Ok(_) = stdin.lock().read_line(&mut buffer) {
                // Пишем считанные строки в сокет
                stream.write_all(buffer.as_bytes()).unwrap();
                // Каждый раз очищаем буфер перед новой порцией данных из стандартно ввода
                buffer.clear();
                // Так же мы можем ожидать данные от сервера на наш запрос
                let mut buf: [u8; 16] = [0; 16];
                // Так как клиент не асинхронный, конечно общение будет поочереди, то есть мы будем ждать
                // ответа от сервера, перед тем как отправить ему новое сообщение.
                let size = stream.read(&mut buf).unwrap();
                let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                println!("Got a message from TCP server:\n{}", message);
            }
        }
        Proto::UDP => {
            // Для клиента создание сокета ничем не отличается от сервера, здесь мы только позволяем операционной системе
            // самостоятельно выбрать порт для клиента
            let socket = UdpSocket::bind("127.0.0.1:0").unwrap();
            let mut buffer = String::new();
            let stdin = io::stdin();
            // Считываем строчки со стандартного ввода (терминала)
            while let Ok(_) = stdin.read_line(&mut buffer) {
                // Пишем считанные строки в сокет
                socket.send_to(buffer.as_bytes(), format!("127.0.0.1:{port}", port=port)).unwrap();
                // Каждый раз очищаем буфер перед новой порцией данных из стандартно ввода
                buffer.clear();
                // Читаем ответ от сервера
                let mut buf: [u8; 16] = [0; 16];
                let (size, _) = socket.recv_from(&mut buf).unwrap();
                let message = std::str::from_utf8(&buf[..size]).unwrap().trim();
                println!("Got a message from UDP server:\n{}", message);
            }
        }
    }
}

fn main() {
    let cli = Cli::parse();

    match cli.mode {
        Mode::Client => {
            println!("Client mode");
            start_client(cli.proto, cli.port);
        }
        Mode::Server => {
            println!("Server mode");
            start_server(cli.proto, cli.port);
        }
    }
}

```


## Тестирование
Начнем с ошибок, если мы просто запустим нашего tcp клиента, то получим:
```bash
./client-server client tcp 65535
Client mode
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Os { code: 61, kind: ConnectionRefused, message: "Connection refused" }', src/main.rs:105:89
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
Это первая особенность tcp сессий, мы не просто запускаем клиента, мы подключаемся к слушающему серверу. А если сервер не
запущен, то мы получаем `Connection refused`. При этом в случае с UDP, все будет работать корректно, так как там мы всего лишь
открываем локальный сокет. Попробуем все же отправить сообщение, первым запускаем сервер:
```bash
./client-server server tcp 65535
Server mode
```
Подключаем клиент:
```bash
./client-server client tcp 65535
Client mode
```
Видим на сервере сообщение о принятии нового соединения:
```bash
New connection from: 127.0.0.1:58535
```
`58535` - это порт, который клиенту автоматически назначила ОС.

Далее на клиенте:
```bash
Hello!
Got a message from TCP server:
Server OK
```

Видим, успешный ответ от сервера. А теперь такой кейс, отправляем с клиента подобное сообщение:
```bash
This message is too big for 16 byte buffer            
Got a message from TCP server:
Server OK
```

В это время на сервере:
```bash
Got a message from TCP client:
This message is
Got a message from TCP client:
too big for 16 b
Got a message from TCP client:
yte buffer
```

То есть, несмотря на ограничение буфера в 16 байт, сообщение не потерялось, а просто порезалось на куски, но дошло
до сервера целиком. А теперь попробуем сделать то же самое при UDP соединении:

```bash
./client-server client udp 65535
Client mode
This message is too big for 16 byte buffer
Got a message from UDP server:
Server OK
```

```bash
./client-server server udp 65535
Server mode
Got message from UDP client:
This message is
```
Как видим, часть сообщения просто пропала.


# Summary
Данное приложение - это демо. Здесь нет обработки ошибок, нет асинхронности, нет нормального логирования. Я постарался 
показать связь современных языков программирования с внутренними механизмами операционной системы. Поэтому заключение
здесь будет такое:
1. Сокет - это как корзина, куда можно скинуть любые данные, либо забрать их оттуда. Остальное - работа сетевого стека
2. TCP - это потоковое соединение, где сообщение либо будет доставлено целиком, либо не будет доставлено вообще
3. UDP быстрее в работе и проще в реализации. Данные считываются в виде каждой датаграммы отдельно, никакой связи между которым нет 
4. Язык программирования дает всего лишь удобную обертку над системными вызовами ОС
5. Системные вызовы могут быть блокирующими и не блокирующими. [Читать](https://habr.com/ru/articles/416669/)
