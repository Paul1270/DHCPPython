Introduction needed to understand and describe listener.py

There are two most popular DHCP clients dhcpcd and dhclient
DHCPCD uses ARP for obtaining lease, it can also use UDP socket. By executing man dhcpd, you can see that this bhaviour can be configured.
DHCP options that are returned by server on query protocol are set in system using HOOKS, hooks are just ordinary shell scripts
vim -lah /usr/lib/dhcpd/dhcpd-hooks/20-resolv.conf this one for example sets DNS server in /etc/resolv.conf on systems that use systemd as init daemon this could 
be not wanted behaviour because systemd can deploy its own DNS relay that could have per interface dns servers or make some filtering.

Based on the above, many network coniguration can be set using dhcp they are described in RFC that's why we introduce listener.py , these options are host_name, name_server
time_server and so on. All of this options are handled using hooks, dhcp client sets basic IP address network mask default route.
These options are used in cluster environments for example where you have many like hundreds of hosts and managing this configuration would be time absorbing good example 
is netboot which can also be configured via dhcp.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Introduction needed to understand and describe dhcp.py

for managing ip address by hand ip address can be used where you specify IP address of host, network mask and interface
ip addr add 172.16.0.33/24 dev enp0s3 after executing this command enp0s3 network card is up with given address and by default route was add to network 172.16.0.0/24
ip route to show routes. So to manage IP bu hand use ip addr, ip route and edit resolv.conf to set dns and to get rid off commands dhcp was introduced.

When you use DHCP client it starts the process that opens one of the sockets udp or arp listener
that process run in background of OS handling thing that is called dhcp transaction that transaction is most important part of every modern dhcp server.
when dhcp server gets client request it basically sends response with IP and other options.
dhcp is time based protocol every IP address is given for some period of time within that time client must perform for example
renew to inform dhcp server that is still alive. One of the attacks described in specification is based on not proper handling transactions,
when clients asks for new address IP address is given to him from a free pool, and client gets transaction ID which i stored server side without that simple program sending bulk packets can
be written that would generate dhcp requests one after another to fill the pool or on some limited resource devices to consume memory or cpu simple example of Denial Of Service. 
One of dhcp options is time for which IP is given to client, as all of the dhcp options it can be used or omitted by client.

Based on the above, in this particular server to handle transaction couple of classes are used:
1-TransactionDelayWorker 
which is used to perform delayed response or request to client checking if it is still alive
2-PriorityQueue
which is sorted binary tree where priority is time, time that is being closer to current timestamp is always on top
3-WriteBootProtocolPacket
DHCP protocol datagram serializer. This class serializes UDP DHCP packet, instance is constructed using global configuration from which dhcp options are copied.
Create new packet instance and search for options set in configuration and copy them to packet
4-DhcpTransaction  
Using this delayed worker and queue that is sorted one thread can be used to make response to dhcp client no matter what 
since os should act as guard here dropping invalid packets only valid ones should get to socket so every new client starts new transaction, if transaction id isnt know
packet is dropped by dhcp server as mentioned transaction handles only dhcpdiscover, request and inform.
important here every client NEW TRANSACTION time to response from server stored in sorted queue with time as sort key

And rest functions talk about packet and their format as this is described in RFC just bits and offsets


5-DHDCPServerConfiguration
Class to load DHCP server configuration command line

        if(len(file) > 0 and exists(file)):
            with open(file) as f:
                exec(f.read(), self.__dict__)
        else:
            args = ' '.join(sys.argv[1:])
            args = re.sub(' -', "\r\n", args)
            args = re.sub('^-', '', args)
            args = re.sub('^([a-z_]+)([ ]+)(.+)$', r"\1=\3", args, flags=re.MULTILINE)
            exec(args, self.__dict__)
	
	
using f.read the contents of the file are read and if the file is formatted in python syntax
some_option="some value"
other_option=666
we can use exec to execute this code in current context so this is configuration file handling without writing a parser for it 
the second one after else is also tricky part we take all of the options passed to server from commandline and make regular expression string substitution
for example if some options are passed:
-option_one value -option_two value2 we change - to \r\n which is line end
so we get such string
option_one value
option_two value2
then remove any - that are left and change 
$1         $3
option_one value
re.sub('^([a-z_]+)([ ]+)(.+)$', r"\1=\3", args, flags=re.MULTILINE)
to $1=$3
option_one=value
option_two=value2
which is python code for variables, so without actually parsing and validating arguments we set all  of the class properties from file or command line this has downside that even invalid option can be passed

5-GREATER ,NETWORK and CASEINSENSITIVE...
used for comparing various items in data sets

6-DHCPServer
Main DHCP server class that is handling incoming packets and sending responses,using all other utility classes

	self.socket = socket(type = SOCK_DGRAM)		    #create datagram (UDP socket)
        self.socket.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1) # set option to reuse address it may happen that after killing process socket is not
								# properly closed and program cant be run without OS closing it but settings this option enables multiple programs to use the same port works on datagram only
        self.socket.bind(('', 67))
        self.delay_worker = TransactionDelayWorker() 		#here is this one worker
        self.closed = False
										#this is time sorted collection of transactions
        self.transactions = collections.defaultdict(lambda: DHCPTransaction(self)) # id: transaction
        self.hosts = HostDatabase(self.configuration.host_file)
        self.time_started = time.time()

7-ThreadedTCPRequestHandler
Control socket client connection handler. Method used to handle client connection parsing commands and giving response to them
