# Chapter 2: How Does the Internet Work? What Are Servers?

## 1. How does the internet work?

Imagine the whole world. You may be sitting in Delhi, Pune, Lucknow or any other place, watching a YouTube video. The video is stored on a computer far away, maybe in the USA. How does it reach you?

### A common wrong idea

Many people think the video comes through a **satellite**. This is mostly not true. Satellites are far away, so data would be slow (high delay, called *latency*).

### The real answer: undersea optical fiber cables

- Data travels through **optical fiber cables**.
- Many of these cables are laid **under the sea** and connect countries and continents.
- These cables connect **data centers** to each other and to your city.

### What is a data center?

A data center is a big building that holds **thousands of computers**. Their job is to:

1. **Store** data.
2. **Send (transmit)** data when someone asks for it.

The data moves through cables.

### Who owns the cables?

Big companies own and manage these fiber cables, for example AT&T and Reliance Jio. They charge you money for using them. What we call an **internet recharge** or **data pack** is basically this payment.

### Your Internet Service Provider (ISP)

Your ISP (for example a broadband company like Airtel Xstream in Pune) connects your home to the wider internet. When you try to open a website, the ISP first checks that you have internet access. Only then does the request go further.

---

## 2. What is a server?

A **server** is simply a **computer whose job is to serve information**. The word "serve" means "to give or deliver".

### Types of servers

| Server type | What it serves |
|-------------|----------------|
| Email server | Emails |
| File server | Stores your files and returns them when needed |
| Database server | Stores data in a database (you insert and read data) |
| Application server | Runs applications such as facebook.com or youtube.com and gives *dynamic* data |
| Web server | Serves *static* content such as images and HTML pages (example: Nginx) |
| DNS server | Converts domain names to IP addresses |

### Web server vs application server (interview question)

- **Web server**: serves **static** data. Static means the content does not change and needs no calculation. Example: images, HTML pages. Tool example: **Nginx**.
- **Application server**: runs **logic and calculations** and gives **dynamic** data. Example: a Django or Node.js application running on a server.

---

## 3. Server vs client (very common interview question)

- **Server**: its work is **to serve information**.
- **Client**: its work is **to request information** from a server.

A client can be your **phone**, your **laptop**, or the **browser** on your laptop.

---

## 4. What happens when you type youtube.com in a browser?

Step by step:

1. Your request goes first to your ISP, which confirms you have internet access.
2. The name `youtube.com` is called a **domain**.
3. Computers do not understand names. They understand **IP addresses**. So the domain must be linked to an IP address.
4. This work is done by a special server called the **DNS server** (**Domain Name Server** / Domain Name System).
5. The DNS server tells the IP address of the application server for that domain.
6. Your request goes to that application server.
7. The server sends back the response (the web page or video).

So: **Browser (client) -> ISP -> DNS -> Server -> Response back to your device.**

---

## 5. Types of applications

### Standalone application

- Does **not need the internet**.
- Does not need a database server, email server, cache, etc. It runs alone.
- Example: a feedback machine at an airport where you press a button, or a coin-operated machine.

### Web application

- Runs on the **internet**.
- Has many supporting parts: email servers, database servers, application servers, and more.
- Example: instagram.com, youtube.com.

### Why should a DevOps engineer know this?

- If you work on a **standalone** app, you rarely need database or network connections.
- If you work on a **web app**, you must handle connections to databases, front end, back end, cloud, and so on.

---

## 6. What is application support and maintenance?

Applications run on an operating system such as Linux, Windows or macOS. They need care, like a patient needs care:

- Are they running properly?
- Are they crashing?
- Is a connection broken (for example, the connection to the email server is lost, so emails are not being sent)?

The work of checking, fixing and improving running applications is called **application support and maintenance**. A DevOps engineer often does this kind of work.

---

## Quick summary

- The internet works through fiber cables (many under the sea) connecting data centers.
- A server serves information; a client requests it.
- A domain name is turned into an IP address by a DNS server.
- Web servers give static data; application servers give dynamic data.
- Applications are either standalone or web applications.
