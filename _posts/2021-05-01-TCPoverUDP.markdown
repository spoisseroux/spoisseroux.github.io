---
layout: post
title:  "[Python] TCP-like reliability transfer over UDP"
date:   2021-05-01 19:48:10
---

**Purpose**

To provide a TCP-like transport protocol over UDP with reliable transfer, relevant headers, and connection establishment.

-----------------------------------------------------------

**Receiver Design**

<img src="https://i.imgur.com/Pd8N9ah.png" width="400">

-----------------------------------------------------------

**Setup** 
(Linux environment required)

+ To begin using our TCP-like Transport Protocol over UDP, please clone this repository into wherever you would like it saved.
+ Open a terminal in the location you would like to receive the file, with the repository cloned.
+ Copy the file you would like to send into the cloned repository's project3/src/.
+ In the receiving terminal, run the following
python3 ./server.py 5000 > RECEIVED_FILE
+ In the sending terminal, run the following
cat FILENAME | python3 ./client.py HOSTNAME-OR-IP 5000
+ Compare the two with:
diff FILENAME RECEIVED_FILE

-----------------------------------------------------------

**Code**

Client.py
    ```python
        #!/usr/bin/env python
        from socket import *
        import sys
        from sys import getsizeof
        import time
        import struct

        #print("first line")

        host = sys.argv[1]
        port = int(sys.argv[2])
        total_kb = 0 #keep track for stats
        buf = 512
        addr = (host,port)
        s = socket(AF_INET,SOCK_DGRAM)
        end_time=0
        seqNum = 42
        conID = 0
        payloadSize = 0
        finished = False
        #s.bind((host,port))

        #start timer
        global start_time
        start_time = time.time()

        def resend(packet):
            #resend lost packet
            global s
            global addr

            flag = ''
            sentSeqNum, sentAck, sentConID, sentFlag, sentPayload = struct.unpack('iihh512s', packet)
            if (sentFlag == 4):
                flag = "ACK"
            elif (sentFlag == 2):
                flag = "SYN"
            elif (sentFlag == 6):
                flag = "ACK SYN"
            elif (sentFlag == 1):
                flag = "FIN"
            elif (sentFlag == 5):
                flag = "ACK FIN"
            print("DROP " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag)
            print("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag + " DUP")

            s.sendto(packet,addr)
            receive(packet) #go to receive lost packet

        def resendSYNACK(packet):
            global s
            global addr

            flag = ''
            sentSeqNum, sentAck, sentConID, sentFlag, sentPayload = struct.unpack('iihh512s', packet)
            if (sentFlag == 4):
                flag = "ACK"
            elif (sentFlag == 2):
                flag = "SYN"
            elif (sentFlag == 6):
                flag = "ACK SYN"
            elif (sentFlag == 1):
                flag = "FIN"
            elif (sentFlag == 5):
                flag = "ACK FIN"
            print("DROP " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag)
            print("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag + " DUP")

            s.sendto(packet,addr)
            receiveSYNACK(packet)

        def receiveSYNACK(packet):
            global s
            global buf
            global host
            global port
            global addr
            global seqNum
            global conID

            try:
                s.settimeout(0.5)
                data, addr = s.recvfrom(buf)
                recvdSeqNum, recvdAck, conID, recvdFlag = struct.unpack('iihh', data)

                flag = ''
                if (recvdFlag == 4):
                    flag = "ACK"
                elif (recvdFlag == 2):
                    flag = "SYN"
                elif (recvdFlag == 6):
                    flag = "ACK SYN"
                elif (recvdFlag == 1):
                    flag = "FIN"
                elif (recvdFlag == 5):
                    flag = "ACK FIN"
                print("RECV " + str(recvdSeqNum) + " " + str(recvdAck) + " " + str(conID) + " " + flag)

                if (recvdFlag != 6 or recvdAck != seqNum+1):
                    resendSYNACK(packet)
            except timeout:
                resendSYNACK(packet)

        def receive(packet, timer=0.5):
            global s
            global buf
            global host
            global port
            global addr
            global seqNum
            global conID
            global payloadSize
            global finished

            #print("in receive()")
            try:
                if (timer != 0.5):
                    s.settimeout(timer)
                else:
                    s.settimeout(timer)
                expectedSeqNum = seqNum
                data,addr = s.recvfrom(buf)
                # TODO split header + payload decoded
                recvdSeqNum, recvdAck, recvdConID, recvdFlag = struct.unpack('iihh', data)
                sentSeqNum, sentAck, sentConID, sentFlag, sentPayload = struct.unpack('iihh512s', packet)

                flag = ''
                if (recvdFlag == 4):
                    flag = "ACK"
                elif (recvdFlag == 2):
                    flag = "SYN"
                elif (recvdFlag == 6):
                    flag = "ACK SYN"
                elif (recvdFlag == 1):
                    flag = "FIN"
                elif (recvdFlag == 5):
                    flag = "ACK FIN"
                print("RECV " + str(recvdSeqNum) + " " + str(recvdAck) + " " + str(recvdConID) + " " + flag)


                if (recvdFlag == 1):
                    seqNum = expectedSeqNum
                    return
                if (sentFlag == 5):
                    seqNum = expectedSeqNum
                    return
                if (sentFlag == 1):
                    return
                    #if ((recvdAck != expectedSeqNum) or recvdConID != conID) or (recvdFlag != 5):
                    #    resend(packet)
                    #seqNum = expectedSeqNum
                    #return
                if (recvdFlag == 0 or recvdFlag == 4):
                    expectedSeqNum = seqNum
                if (expectedSeqNum >= 204801): #after 204800 is zero
                    expectedSeqNum = abs(204800 - expectedSeqNum)
                #check if received correct data (ACK)
                #print("receive if state (recvdSeq, seqNum, recvdAck, expected SeqNum)" + str(recvdSeqNum) + " " + str(seqNum) + " " +  str(recvdAck) + " " + str(expectedSeqNum))
                if ((recvdAck != expectedSeqNum)
                    or (recvdConID != conID) or (recvdFlag != 4)):
                    #print("if final\n)")
                    resend(packet)
            except timeout: #ACK got lost
                if (timer == 0.5):
                    resend(packet)
            seqNum = expectedSeqNum

        def send(packet):
            global s
            global total_kb
            global host
            global port
            global buf
            global addr
            global seqNum

            flag = ''
            sentSeqNum, sentAck, sentConID, sentFlag, sentPayload = struct.unpack('iihh512s',packet)
            if (sentFlag == 4):
                flag = "ACK"
            elif (sentFlag == 2):
                flag = "SYN"
            elif (sentFlag == 6):
                flag = "ACK SYN"
            elif (sentFlag == 1):
                flag = "FIN"
            elif (sentFlag == 5):
                flag = "ACK FIN"
            print("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag)

            s.sendto(packet,addr)
            total_kb += buf #track kb sent

        def main():
            #get data, call send, recv after getting data in loop
            global buf
            global end_time
            global seqNum
            global payloadSize
            global finished

            #print("in getdata() init")

            completeHandshakePacket = handshake()
            receive(completeHandshakePacket)
            data = sys.stdin.read(buf)
            #print("got data")
            packet = buildStandardHeader(data)
            seqNum = seqNum + payloadSize
            #print(seqNum)

            try:
                while (data):
                    #print("in getdata() while loop")
                    s.settimeout(10)
                    #seqNum = seqNum + 1
                    send(packet) #send data
                    receive(packet) #receive ack for sent data
                    #seqNum = seqNum + payloadSize
                    if (seqNum >= 204800): #make sure seqnum never goes above 9
                        seqNum = abs(204800 - seqNum) #TODO IS THIS RIGHT?
                    data = sys.stdin.read(buf)
                    packet = buildStandardHeader(data)
                    seqNum = seqNum + payloadSize
                packet = buildFinHeader()
                send(packet)
                receive(packet)
                finTimerStart = time.time()
                finTimerDiff = time.time()
                while ((finTimerDiff - finTimerStart) <= 2):
                    finTimerDiff = time.time()
                    receive(packet, 2)
                    packet = buildFinAckHeader()
                    send(packet)
                # try w/ timeout 2
                # receive ack from server
                # receive fin's from server and ack em

            except timeout:
                s.close()
                sys.exit(1)
            s.close()
            sys.exit(0)

        def printStats():
            #end timer

            elapsed_time = end_time - start_time

            #statistics
            if (elapsed_time == 0):
                elapsed_time = 0.001
            kbRate = (total_kb / 125) / elapsed_time
            kbRate = float("%0.2f" % (kbRate))
            elapsed_time = float("%0.3f" % (elapsed_time))

        def buildStandardHeader(payload):
            global conID
            global seqNum
            global payloadSize

            payloadSize = len(payload)
            payload = payload.encode()
            header = struct.pack('iihh512s', seqNum, 0, conID, 0, payload)

            return header

        def buildFirstHandshakeHeader():
            global seqNum
            emptyPayload = bytearray(512)
            header = struct.pack('iihh512s', seqNum, 0, 0, 2, emptyPayload)
            #print("FIRST HANDSHAKE HEADER")
            return header

        def buildThirdHandshakeHeader(payload):
            global conID
            global seqNum
            global payloadSize

            #payload = bytes(payload, 'utf-8')

            payload = payload.encode()
            payloadSize = len(payload)
            header = struct.pack('iihh512s', seqNum, 0, conID, 4, payload)
            #print("THIRD HANDSHAKE HEADER")
            return header

        def buildFinHeader():
            global conID
            global seqNum

            emptyPayload = bytearray(512)
            header = struct.pack('iihh512s', seqNum, 0, conID, 1, emptyPayload)
            #print("FIN HEADER")
            return header

        def buildFinAckHeader():
            global conID
            global seqNum

            emptyPayload = bytearray(512)
            header = struct.pack('iihh512s', seqNum, seqNum + 1, conID, 5, emptyPayload)
            #TODO WHAT IS SEQNUM
            #print("FIN ACK HEADER")
            return header

        def handshake():
            global seqNum
            initpacket = buildFirstHandshakeHeader()
            send(initpacket)
            #print("init" + str(seqNum))
            receiveSYNACK(initpacket)
            data = sys.stdin.read(buf)
            seqNum = seqNum + 1
            secondpacket = buildThirdHandshakeHeader(data)
            send(secondpacket)
            #print("init2" + str(seqNum) + " data " + str(data))
            seqNum = seqNum + payloadSize
            return secondpacket

        main()
    ```

Server.py
<details>
    ```python
    from socket import *
    import sys, os
    import select
    import struct
    import random

    host = "127.0.0.1"
    port = int(sys.argv[1])
    s = socket(AF_INET,SOCK_DGRAM)
    s.bind((host,port))
    addr = (host,port)
    buf=524
    seqNum = 42
    conID = 0
    payloadSize = 0
    prevFlag = 0
    recvdFlag = 0
    prevSize = 1
    prevSeq = 0
    firstHandshakeResend = False

    def recieve():
        global s
        global buf
        global addr
        global seqNum
        global conID
        global prevFlag
        global recvdFlag
        global prevSize
        global prevSeq
        global firstHandshakeResend

        data,addr = s.recvfrom(buf)
        prevFlag = recvdFlag
        recvdPayload = struct.unpack_from('512s', data, 12)[0]
        recvdPayload = recvdPayload.decode().rstrip('\x00')
        recvdSeqNum, recvdAck, recvdConID, recvdFlag, recvdPayloadNull = struct.unpack('iihh512s', data)

        flag = ''
        if (recvdFlag == 4):
            flag = "ACK"
        elif (recvdFlag == 2):
            flag = "SYN"
        elif (recvdFlag == 6):
            flag = "ACK SYN"
        elif (recvdFlag == 1):
            flag = "FIN"
        elif (recvdFlag == 5):
            flag = "ACK FIN"
        recvPrint = "RECV " + str(recvdSeqNum) + " " + str(recvdAck) + " " + str(recvdConID) + " " + flag

        #recvdPayload = recvdPayload.decode()
        if (prevFlag == 1 and recvdFlag == 5):
            s.close()
            sys.exit(0)
        if (recvdFlag == 2): # first handshake
            if (firstHandshakeResend == True):
                sys.stderr.write("DROP " + str(recvdSeqNum) + " " + str(recvdAck) + " " + str(recvdConID) + flag + "\n")
                seqNum = seqNum - 1
                recvPrint = recvPrint + " DUP"
            conID = random.randint(1, 20)
            prevSize = 1
            firstHandshakeResend = True
        # TODO elif recvdFlag ==4 and prevFlag == 1
        sys.stderr.write(recvPrint + "\n")
        if (recvdFlag == 1):
            finning()
        elif ((recvdFlag == 4 or recvdFlag == 0) and (conID == recvdConID)):
            if (recvdSeqNum == seqNum):
                sys.stdout.write(recvdPayload)
            elif (recvdSeqNum == prevSeq):
                seqNum = prevSeq
                sys.stderr.write("DROP " + str(recvdSeqNum) + " " + str(recvdAck) + " " + str(recvdConID) + flag + "\n")
                # retransmission
            prevSize = len(recvdPayload)
        # elif recvdFlag == 1 TODO
        prevSeq = seqNum
        # TODO prevFlag = recvdFlag

    def finning():
        global addr
        global prevFlag
        global recvdFlag

        packet = buildFinAckHeader()

        flag = ''
        sentSeqNum, sentAck, sentConID, sentFlag = struct.unpack('iihh', packet)
        if (sentFlag == 4):
            flag = "ACK"
        elif (sentFlag == 2):
            flag = "SYN"
        elif (sentFlag == 6):
            flag = "ACK SYN"
        elif (sentFlag == 1):
            flag = "FIN"
        elif (sentFlag == 5):
            flag = "ACK FIN"
        sys.stderr.write("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag + "\n")

        s.sendto(packet, addr)
        #recv all the stuff
        packet = buildFinHeader()

        flag = ''
        sentSeqNum, sentAck, sentConID, sentFlag = struct.unpack('iihh', packet)
        if (sentFlag == 4):
            flag = "ACK"
        elif (sentFlag == 2):
            flag = "SYN"
        elif (sentFlag == 6):
            flag = "ACK SYN"
        elif (sentFlag == 1):
            flag = "FIN"
        elif (sentFlag == 5):
            flag = "ACK FIN"
        sys.stderr.write("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag + "\n")

        s.sendto(packet,addr)
        recieve()

    def buildFinHeader():
        global conID
        global seqNum

        header = struct.pack('iihh', seqNum, 0, conID, 1)

        return header

    def buildFinAckHeader():
        global conID
        global seqNum

        ackNum = seqNum + prevSize
        header = struct.pack('iihh', seqNum, ackNum, conID, 5)
        return header

    def send():
        global s
        global total_kb
        global host
        global port
        global buf
        global addr
        global recvdFlag
        global prevSize
        global conID
        global seqNum

        ackNum = seqNum + prevSize
        if (ackNum >= 204800): ackNum = abs(204800 - ackNum)

        #dummyPayload = bytearray(512)
        if (recvdFlag == 2):
            packet = struct.pack('iihh', seqNum, ackNum, conID, 6)
        elif (recvdFlag == 1): #final
            pass #TODO
            # define and send ack
            # packet = defined fin
        #elif recvdFlag == 4 and prev == 1 then return
        elif (recvdFlag == 0 or recvdFlag == 4):
            packet = struct.pack('iihh', seqNum, ackNum, conID, 4)

        flag = ''
        sentSeqNum, sentAck, sentConID, sentFlag = struct.unpack('iihh', packet)
        if (sentFlag == 4):
            flag = "ACK"
        elif (sentFlag == 2):
            flag = "SYN"
        elif (sentFlag == 6):
            flag = "ACK SYN"
        elif (sentFlag == 1):
            flag = "FIN"
        elif (sentFlag == 5):
            flag = "ACK FIN"
        sys.stderr.write("SEND " + str(sentSeqNum) + " " + str(sentAck) + " " + str(sentConID) + " " + flag + "\n")

        s.sendto(packet,addr)

    def main():
        global seqNum #init seqnum
        global prevSize

        try:
            while(True):
                s.settimeout(2)
                recieve()
                send()
                seqNum = seqNum + prevSize #increase next expected seqNum
                if (seqNum >= 204800): seqNum = abs(204800 - seqNum)

        except timeout:
            #end
            s.close()
            sys.stderr.write("File received, exiting.\n")
            pass

    main()
    ```
</details>

[[repo]](https://github.com/spoisseroux/CS488S21PROJS/tree/main/project3)

