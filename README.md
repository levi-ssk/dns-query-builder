In this project, I will implement a DNS client that can query an existing DNS server for domain name to IP
address translation. Nslookup and dig are two tools that can be used to achieve the same translation using
command line. However, in this programming assignment, I will manually handcraft DNS query messages,
send them to known DNS servers, and process their response.  
My DNS client has three big high-level pieces (in order of operations):

1. DNS query: the client should read the hostname provided by a user and prepare a query message which adheres to DNS protocol specifications. Note that if my message does not adhere to the specifications, DNS server will not be able to identify/understand my query.
2. Send query: the client should create a UDP socket connection to the server and send the DNS query message I prepared above.
3. Receive and process DNS response: the client should receive a response from the DNS server. It should then process the response based on the format of DNS response message. It should extract the necessary information (IP address and more) and display it to the user on command line.  

# DNS query

My client should be able to read the hostname provided by a user. The hostname will be provided as a command line argument to my client program as follows `$> my-dns-client <host-name>`.  An example of this would be `$> my-dns-client google.com`. Although a typical DNS client can query multiple hostnames at the same time, we will restrict to only one hostname as an argument in this project. Once my client program reads the hostname, it should prepare a DNS query message. Specific bit-by-bit format of the query message can be found in the official DNS RFC document (https://tools.ietf.org/html/rfc1035#page-25).

The query and response messages have 5 sections shown in Fig. 1. my query message will only first two sections.
![image](https://github.com/levi-ssk/dns-builder/assets/156009236/48a88745-6061-48fd-93f2-27d644558a07)

![image](https://github.com/levi-ssk/dns-builder/assets/156009236/ec30e020-295e-4f37-a447-da82fb833388)

The header contains the following fields (refer to Fig.2). We have described only the important fields here.

- ID: A 16 bit id that can uniquely identify a query message. Note that when I send multiple query messages, I will use this ID to match the responses to queries. The ID can be randomly generated by my client program.
- QR: 0 for query and 1 for response.
- OPCODE: We are only interested in standard query.
- RD: This bit is set if client wants the name server to recursively pursue the query. We will set this to 1.
- QDCOUNT: Since we are only sending one hostname query at a time, we will set this to 1.

  
The question section has the following format (more details available from the RFC draft).
![image](https://github.com/levi-ssk/dns-builder/assets/156009236/38b73889-e729-43d9-bb4c-e610b09b3f06)

- QNAME: it contains multiple labels - one for each section of a URL. For example, for gmu.edu, there should be two labels for gmu and edu. Each label consists of a length octet (3 for gmu), followed by ASCII code octets (67 for g, 6D for m, 75 for u). This is repeated for each label of the URL. The QNAME terminates with the zero length octet for the null label of the root. Note that this field may be an odd number of octets; no padding is used.
- QTYPE: Set to 1 because we are only interested in A type records.

# Send Query

Once I have prepared a query message, my client program will establish a socket connection with a DNS server. We will use Google’s public DNS server in the project. Learn about public DNS server at https://www.lifewire.com/free-and-public-dns-servers. The IP address of Google’s public DNS server is 8.8.8.8.
DNS query messages sent over UDP can be lost in the network. My code will implement a timeout based retry where it waits for 5 seconds and if the server does not respond within that time, it resend the query message. If no response is received after 3 attempts, this should print out an error message about timeout.

# Receive and process response

After sending the DNS query to server, the server will reply back. Assuming my query is properly formatted and the server is able to understand my request, the response will follow standard DNS response message format. If my query message is not properly formatted, I might receive an error message in response or not receive any reply at all!
A response can have all five sections shown in Fig.1. First, let us assume that this is a standard response, in which case, it will have top three sections (header, question and answer).  
The header section will be similar to that of DNS query.

- ID will be the same as the query.
- QR will be set to 1 because it is a response.
- AA will depend on if the server is authority for the domain or not (most likely it is not, so AA is 0).
- RCODE: response code will be 0 if no errors, and >0 otherwise.  
As with the query format, make sure to go over the RFC specification to ensure each field is properly interpreted by my client.  
The question section will be exactly the same as the query message.

The answer section will include one or more resource records (RRs). An RR has the following format:  
![image](https://github.com/levi-ssk/dns-builder/assets/156009236/bd8ba185-8a09-4f07-9927-0a786134c297)

Some of the important fields are:
- NAME is the domain for which the IP address is resolved. It uses a compressed format which can be ignored in my processing.
- TYPE and CLASS are the same as query messages.
- TTL (Time To Live): specifies the time interval (in seconds) that the RR may be cached before considered outdated.
- RDATA is the resolved IP address.

After my client program has parsed and processed the request, my client program should print <field,value> tuple for every field of the response message. This would look something like:

```$> my-dns-client google.com  
Preparing DNS query..
Contacting DNS server..
Sending DNS query..
DNS response received (attempt 1 of 3)
Processing DNS response.. ----------------------------------------------------------------------------
header.ID = <value>
header.QR = <value>
header.OPCODE = <value>
….
….
question.QNAME = <value>
question.QTYPE = <value>
5
question.QCLASS = <value>
….
….
answer.NAME = <value>
answer.TYPE = <value>
….
….
answer.RDATA = <value>

…. (include any authority or additional RRs received in the response also) ----------------------------
```
