there are two most popular dhcp clients on linux hosts:
dhcpcd 
dhclient

dhcpcd uses ARP for obtaining lease, it can also use udp socket and IPv4LL last one is zero configuration 
	protocol used by windows and mac rather by linux 

//Command: man dhcpd
as you can see in man this behaviour can be configured 

//command: ls -lah /usr/lib/dhcpd/dhcpd-hooks/
dhcp options that are returned by server on query protocol are set in system using HOOKS, hooks are just ordinary shell scripts

//command: vim -lah /usr/lib/dhcpd/dhcpd-hooks/20-resolv.conf
this one for example sets DNS server in /etc/resolv.conf on systems that use systemd as init daemon this could 
be not wanted behaviour because systemd can deploy its own DNS relay that could have per interface dns servers or make some filtering

//command: cat /etc/resolv.conf
this configuration is on arch install media look at the dns namseserver address which is 127.0.0.53
127.0.0.1 is loopback address it is virtual address of localhost

//command: ifconfig or ip addr or ip package
to check ip address of linux machine 
as you can see there are two network cards one is lo and second is enp0s3
127.0.0.1/8 means first 8 bytes are defining network and rest is host address in network
so every address begining with 127 is treated as private not routable address
even trying to use firewall or other nat/snat/dnat utils you cant forward connections from outer network to loopback

//command: cat /run/systemd/resolve/resolv.conf
so as you can see networkmanager has its own files for managing dns serversr
by default it uses 1.1.1.1 which is cloudflare dns

//listener.py --> section options
as I said before many network coniguration can be set using dhcp they are described in RFC. this options are host_name, name server, time server and so on all of this options
are handled using hooks dhcp client sets basic IP address network mask default route
this options are used in cluster environments like slurm for example where you have many like hundreds of hosts
and managing this configuration would be time absorbing good example is netboot which can also be configured via dhcp
dhcp server sends address of next server and tftp trivial ftp which works on UDP
for sending operating system kernel and initial ramdisk that is used to boot setup network and then mount some root filesystem on NFS or other network filesystem


for managing ip address by hand ip address can be used where you specify IP address of host, network mask and iface

//command: ip addr add 172.16.0.33/24 dev enp0s3
now after executing this command enp0s3 network card is up with given address
and by default route was add to network 172.16.0.0/24 first 3 octets are network address [172.16.0](.0) [network](host) so broacast address for this network is 172.16.0.255
it can also be 255.255.255.255 but this would send broadcast packet to all networks not just for the wanted one 172.16.0.255 

//command: ip route	
as you can see route was added informing OS that through enp0s3 network 172.16.0.0 can be reached,this does not give internet access to.
to get internet access which is external network you must inform OS where to route packets, this is done using default route

//command: ip route add default via 172.16.0.1 dev enp0s3
after this command OS is informed to get access to external networks it must route packets through
172.16.0.1 which should be a router address that is making ip forwarding or masquerade for private networks
private networks are 10.0.0.0, 192.168.0.0 and 172.16.0.0 they have different network mask

//command: ip route
as you can see now you have entry called default this gives internet access.

last thing that is needed to be set is DNS servers
for example by editing resolv.conf file and adding this entry there
on linux mac and even windows there is also hosts file with name <-> ip lookup table

one important thing Windows, linux, mac have most of their IP stack copied from BSD which is unix reasmble

to manage IP bu hand use ip addr, ip route and edit resolv.conf to set dns and to get rid off commands dhcp was introduced
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
**DHCP Explanation
when you use dhcp client it starts the process that opens one of the sockets udp, LL, or arp listener
that process run in background of OS handling thing that is called dhcp transaction that transaction is most important part of every modern dhcp server.

why is dhcp transaction so important? 
when dhcp server gets client request it basically sends response with IP and other options there is some negotiation but thus could be described later on.
dhcp is time based protocol every IP address is given for some period of time within that time client must perform for example
renew to inform dhcp server that is still alive one of the attacks described in specification is based on not proper
handling transactions, when clients asks for new address IP address is given to him from a free pool,
and client gets transaction ID which i stored server side without that simple program sending bulk packets can
be written that would generate dhcp requests one after another to fill the pool or on some limited resource devices to consume memory or cpu 
simple example of Denial Of Service. 
One of dhcp options is time for which IP is given to client, as all of the dhcp options
it can be used or omitted by client.

//command: cat/etc/dhcpcd.conf
as you can see here dhclient requests domain name servers, domain_name, search, static routes,host name and not request 
for example ntp server dhclient has lease time in configuration file set by default and doesnt care by the one server offers
	
in this particular server to handle transaction couple of classes are used:
1-TransactionDelayWorker 
which is used to perform delayed response or request to client checking if it is still alive
2-PriorityQueue
which is sorted binary tree where priority is time, time that is being closer to current timestamp is always on top
3-DhcpTransaction  
Using this delayed worker and queue that is sorted one thread can be used to make response to dhcp client no matter what 
since os should act as guard here dropping invalid packets only valid ones should get to socket so every new client starts new transaction, if transaction id isnt know
packet is dropped by dhcp server as you can see transaction handles only dhcpdiscover, request and inform
important here every client NEW TRANSACTION time to response from server stored in sorted queue with time as sort key

there is nothing intresting to talk about packet and their format as this is described in RFC just bits and offsets --> confirmed nothing important



4-DHDCPServerConfiguration
next class in server is server configuration here I used trick which is known only by experienced python users

        if(len(file) > 0 and exists(file)):
            with open(file) as f:
                exec(f.read(), self.__dict__)
        else:
            args = ' '.join(sys.argv[1:])
            args = re.sub(' -', "\r\n", args)
            args = re.sub('^-', '', args)
            args = re.sub('^([a-z_]+)([ ]+)(.+)$', r"\1=\3", args, flags=re.MULTILINE)
            exec(args, self.__dict__)
	
	
using fread the contents of the file are read and if the file is formatted in python syntax
some_option="some value"
other_option=666
we can use exec to execute this code in current context
so this is configuration file handling without writing a parser for it 
the second one after else is also tricky part we take all of the options passed to server from commandline and make regukar expression string substitution
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
which is python code for variables, so without actually parsing and validating arguments we set all 
of the class properties from file or command line this has downside that even invalid option can be passed

5-GREATER(object) and NETWORK and CASEIN.... and ALL 
used for comparing various items in data sets

6-DHCPServer
	self.socket = socket(type = SOCK_DGRAM) #create datagram (UDP socket)
        self.socket.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1) # set option to reuse address it may happen that after killing process socket is not
								# properly closed and program cant be run without OS closing it but settings this option enables multiple programs to use the same port works on datagram only
        self.socket.bind(('', 67))
        self.delay_worker = TransactionDelayWorker() #here is this one worker
        self.closed = False
										#this is time sorted collection of transactions
        self.transactions = collections.defaultdict(lambda: DHCPTransaction(self)) # id: transaction
        self.hosts = HostDatabase(self.configuration.host_file)
        self.time_started = time.time()



in order dhcp to work network interface on server side must be up and have valid ip address
it can be bring up using ip addr command 
i use cat to quickly view file content

ip addr add 192.168.137.1/24 dev enp0s3
this adds ip address with netmask /24 255.255.255.0 on enp0s3


just i have quick question why you add IP? what is the purpose

on server network card MUST HAVE network address, without it, it must work in promiscous mode
which is out of topic now, any basic dhcp usage requires ip addr on network interface
noted cant work on network interface without ip addr in most cases it can be described like that:
when network interface has Ip address network route is added to kernel routing table when some data arrive on socket and 
is sent back this route table is used, so without ip addr kernel doesnt know from which interface to send
back back in 10 min
okay and you added 192.168.137.1/24 , could i add for example 176.24.6.8/16 or no?
well on network not connected to internet yes
even on that connected to internet this would just mask IP address which server have
think of VPN connection
[ Network   ]                                              [ Network     ]
[           ]  <========VPN====l2tp/sstp/ovpn etc.=======> [ DHCP server ]
[ Segment   ]                                              [ Segment     ]

Network segment connect through VPN connection uses dhcp in second network segment okkk

Now why did I made control socket for commands on tcp connection

class ThreadedTCPServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    """DHCP server control interface TCP server
    """
    def setEvents(self,data):
        """Set DHCP events dictionary reference
        """
        self.events = data

    def setHosts(self,data):
        """Set DHCP host database with active leases reference
        """
        self.hosts = data
        
    def setConfiguration(self,data):
        """Set DHCP UDP global configuration reference
        """
        self.configuration = data

class ThreadedTCPRequestHandler(socketserver.StreamRequestHandler):
    """Control socket client connection handler
    """
    def handle(self):
        """Method used to handle client connection parsing commands and giving response to them
        """
        self.request.sendall(bytes("Welcome to micro python dhcp server", 'ascii'))
        try:
            while(True):
                self.request.sendall(bytes("\r\npydhcp ?> ", 'ascii'))
                data = self.rfile.readline().strip()
                if(data.decode() == "hosts"):
                    self.request.sendall(bytes("Active Hosts:\r\n{}".format("\r\n".join(self.server.hosts.all())),'ascii'))
                elif(data.decode() == "events"):
                    self.request.sendall(bytes("Events last 24h:\r\n{}".format("\r\n".join(map(str,self.server.events.items()))),'ascii'))
                elif(data.decode() == "configuration"):
                    self.request.sendall(bytes("Current configuration\r\n", 'ascii'))
                    for value in options:
                        if(hasattr(self.server.configuration,value[0])):
                            self.request.sendall(bytes("{}: {}\r\n".format(value[0],getattr(self.server.configuration,value[0])),'ascii'))
                elif(data.decode() == "help"):
                    self.request.sendall(bytes("hosts\t\tdisplay host database\r\n",'ascii'))
                    self.request.sendall(bytes("events\t\tdisplay DHCP event log\r\n",'ascii'))
                    self.request.sendall(bytes("configuration\tdisplay current server configuration\r\n",'ascii'))
                    self.request.sendall(bytes("help\t\tthis command\r\n",'ascii'))
                    self.request.sendall(bytes("quit\t\tdisconnect from current session\r\n",'ascii'))
                elif(data.decode() == "quit"):
                    self.request.sendall(bytes("bye\r\n", 'ascii'))
                    break
                else:
                    self.request.sendall(bytes("unknown command: {}".format(data.decode('ascii')), 'ascii'))
        except Exception as e:
            pass


Because without gui another option is adding command parser in main program thread but this let to get server options 
only from local machine this involves as much code as this solution 


 while(True):
                self.request.sendall(bytes("\r\npydhcp ?> ", 'ascii'))
                data = self.rfile.readline().strip()
                if(data.decode() == "hosts"):
                    self.request.sendall(bytes("Active Hosts:\r\n{}".format("\r\n".join(self.server.hosts.all())),'ascii'))
                elif(data.decode() == "events"):
                    self.request.sendall(bytes("Events last 24h:\r\n{}".format("\r\n".join(map(str,self.server.events.items()))),'ascii'))
                elif(data.decode() == "configuration"):
                    self.request.sendall(bytes("Current configuration\r\n", 'ascii'))
                    for value in options:
                        if(hasattr(self.server.configuration,value[0])):
                            self.request.sendall(bytes("{}: {}\r\n".format(value[0],getattr(self.server.configuration,value[0])),'ascii'))
                elif(data.decode() == "help"):
                    self.request.sendall(bytes("hosts\t\tdisplay host database\r\n",'ascii'))
                    self.request.sendall(bytes("events\t\tdisplay DHCP event log\r\n",'ascii'))
                    self.request.sendall(bytes("configuration\tdisplay current server configuration\r\n",'ascii'))
                    self.request.sendall(bytes("help\t\tthis command\r\n",'ascii'))
                    self.request.sendall(bytes("quit\t\tdisconnect from current session\r\n",'ascii'))
                elif(data.decode() == "quit"):
                    self.request.sendall(bytes("bye\r\n", 'ascii'))
                    break
                else:
                    self.request.sendall(bytes("unknown command: {}".format(data.decode('ascii')), 'ascii'))
        except Exception as e:
            pass

This is simple as fuck command parser
this commands can be send to server on control socket.
















--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------