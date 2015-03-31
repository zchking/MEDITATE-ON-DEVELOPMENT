 Channels in Go
================

Goroutines allow you to run a piece of code in parallel to others. But to employ it usefully, there are a few additional requirements - we should be able to pass data into the running process and we should be able to get data out of the running process when it is done creating it. Channels provide the way to do that, and they work alongside goroutines.

A channel can be imagined to be a pipe or a conveyer belt of a defined size and capacity. On one side one can place an item onto this imaginary conveyer belt and on the other side one can take it.


We’ll use the example of a cake making and cake packing factory. There is a machine that can make a cake, and another that puts it into a box. They communicate via this conveyer belt - the maker puts a cake on the conveyer belt, the packer takes a cake off it when it finds one and puts it in a box.

In Go, the chan keyword is used to define a channel. The make keyword is used to create it, along with the type of data that the channel can hold.

Partial code::


    ic := make(chan int) //a channel that can send and receive an int
    sc := make(chan string) //a channel that can send and receive a string
    myc := make (chan my_type) //a channel for a custom defined struct type


You can indicate the sending or receiving of data on the channel by using the operator <- before or after the variable name of the channel. If my_channel is a channel that takes an int, you can send data on the channel with my_channel <- 5, and you can receive the value in a variable with my_recvd_value <- my_channel. Imagine the channel to be the conveyer belt to know the direction to use: an arrow pointing into the channel puts data into it and an arrow pointing out takes data out of it.

Partial code::


    my_channel := make(chan int)

    //within some goroutine - to put a value on the channel
    my_channel <- 5 

    //within some other goroutine - to take a value off the channel
    var my_recvd_value int
    my_recvd_value = <- my_channel


You can also specify the direction of data movement on a channel by indicating the direction around the chan keyword. We shall see uses for this later.

Partial code::


    ic_send_only := make (<-chan int) //a channel that can only send data - arrow going out is sending
    ic_recv_only := make (chan<- int) //a channel that can only receive a data - arrow going in is receiving 


The number of items that the channel (which is our conveyer belt) can hold is important. It indicates how many items can be worked with at a time. Even if the sender is capable of producing multiple items, if the receiver is not capable of accepting them, then it won’t work. There will be too many cakes falling off the conveyer belt and getting wasted. (I wasn’t very happy with my conveyer belt metaphor since our channel isn’t actually moving - but it worked well to explain the falling cakes!) In parallel computing, this is called the producer-consumer synchronization problem.

 If the capacity of the channel is 1 - i.e. once an item is placed on the channel, it has to be taken off before another one is put in its place, this becomes a synchronous channel. Each side - the sender and the receiver - is communicating one item at a time, and has to wait until the other side performs either a sending or a receiving correspondingly. We will start working with synchronous channels.

All the channels we have defined until now defaults to a synchronous channel i.e. a datum put on the channel has to be taken off before another one is placed on it. Let’s implement our cake making and packing factory now. Since channel communication happens between goroutines, there are two aptly named functions makeCakeAndSend and receiveCakeAndPack. Each receive the same reference to a channel as a parameter so that they may communicate using it.

Full code::

    package main

    import (
        "fmt"
        "time"
        "strconv"
    )

    var i int

    func makeCakeAndSend(cs chan string) {
        i = i + 1
        cakeName := "Strawberry Cake " + strconv.Itoa(i)
        fmt.Println("Making a cake and sending ...", cakeName)
        cs <- cakeName //send a strawberry cake
    }

    func receiveCakeAndPack(cs chan string) {
        s := <-cs //get whatever cake is on the channel
        fmt.Println("Packing received cake: ", s)
    }

    func main() {
        cs := make(chan string)
        for i := 0; i<3; i++ {
            go makeCakeAndSend(cs)
            go receiveCakeAndPack(cs)

            //sleep for a while so that the program doesn’t exit immediately and output is clear for illustration
            time.Sleep(1 * 1e9)
        }
    }


    Making a cake and sending ... Strawberry Cake 1
    Packing received cake: Strawberry Cake 1
    Making a cake and sending ... Strawberry Cake 2
    Packing received cake: Strawberry Cake 2
    Making a cake and sending ... Strawberry Cake 3
    Packing received cake: Strawberry Cake 3

In the code above, we make three calls to make a cake and immediately after that to pack it. We know then that there will be one cake ready by the time we perform a call to pack it. The code is slightly fudged though - there is a time lag between the call to print "Making a cake and sending …" and the actual sending of the cake to the channel. The time.Sleep() call we have made in each loop causes a pause that gives the effect that the making and packing is happening one after the other - that effect is correct. Since our channel is synchronous and allows only one item at a time, a removal from the channel and a packing has to happen before making a new cake and putting it on the channel.

Let’s change the code now as we progress to make it a little more like code we would normally use. Typically goroutines could be blocks of code that run repeatedly within itself, performing operations and exchanging data with other goroutines via channels. In this next example, we move the loop inside goroutine functions and we make a call to the goroutine only once. In the interest of time and for illustration, we make only 3 cakes and pack it.

Full code::

package main                                                                                                                                                           

import (
    "fmt"
    "time"
    "strconv"
)

func makeCakeAndSend(cs chan string) {
    for i := 1; i<=3; i++ {
        cakeName := "Strawberry Cake " + strconv.Itoa(i)
        fmt.Println("Making a cake and sending ...", cakeName)
        cs <- cakeName //send a strawberry cake
    }   
}

func receiveCakeAndPack(cs chan string) {
    for i := 1; i<=3; i++ {
        s := <-cs //get whatever cake is on the channel
        fmt.Println("Packing received cake: ", s)
    }   
}

func main() {
    cs := make(chan string)
    go makeCakeAndSend(cs)
    go receiveCakeAndPack(cs)

    //sleep for a while so that the program doesn’t exit immediately
    time.Sleep(4 * 1e9)
}


Making a cake and sending ... Strawberry Cake 1
Making a cake and sending ... Strawberry Cake 2
Packing received cake: Strawberry Cake 1
Packing received cake: Strawberry Cake 2
Making a cake and sending ... Strawberry Cake 3
Packing received cake: Strawberry Cake 3

The output above is as it is on my computer. Yours could vary depending on the execution of the goroutines on your machine. As was mentioned, we are calling each of the goroutines only once and passing it the common channel. Within each goroutine there are three loops making three cakes, the makeCakeAndSend putting it on the channel and receiveCakeAndPack taking it off the channel. Because the program would finish immediately after making the two goroutine calls, we have to artificially add a timer to pause it for a while until three cakes are made and packed.

It is important that we understand that the output as shown is not the correct reflection of the actual sending and receiving on the channel. The sending and receiving here is synchronous - one cake at a time. However due to the time lag between the print statement and the actual channel sending and receiving, the output seems to indicate an incorrect order. So in reality what is happening is:


Making a cake and sending ... Strawberry Cake 1
Packing received cake: Strawberry Cake 1
Making a cake and sending ... Strawberry Cake 2
Packing received cake: Strawberry Cake 2
Making a cake and sending ... Strawberry Cake 3 Packing received cake: Strawberry Cake 3

So do remember to be careful about analyzing code with printed logs when dealing with goroutines and channels. 
