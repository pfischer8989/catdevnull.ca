---
layout: post
title:  "Python and Go programs for listing all tcp sockets and services"
date:   2017-06-19 21:17:21 -0400
categories: jekyll update
---

I wrote two small programs to go through tcp ports 23 to 100 and list the service associated with them. The first one is in Python and the second one is in Go.  

Python

{% highlight python %}

#!/usr/bin/python

import socket

def serviceName():

   for x in range(23, 100):
       try:
         protocol = 'tcp'
         print x
         service = socket.getservbyport(x, protocol)
       except socket.error, err_msg:
           print "%s: %s" %(x, err_msg)

       print "%s" %(service)

   print "service is", service


if __name__ == '__main__':
    serviceName()

{% endhighlight %}



Go

{% highlight go %}

package main

import (
    "fmt"

    "honnef.co/go/netdb"
)

/* func LookupPort(network, service string) (port int, err error)

func GetServByPort(port int, protocol *Protoent) *Servent
*/
func main() {

    var proto *netdb.Protoent
    var serv *netdb.Servent

    proto = netdb.GetProtoByName("tcp")

    for x := 23; x <= 100; x++ {

        serv = netdb.GetServByPort(x, proto)
        if serv != nil {

            fmt.Println("Service name : ", serv.Name)
            fmt.Println("Port number : ", x)
        }
    }
    /*
        if err != nil {
            fmt.Println("Error: ", err.Error())
            os.Exit(2)
        }
    */
}


{% endhighlight %}



Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
