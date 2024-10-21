# Just Radio Transmission stuff

# Rationale
There is no good way as of writing to transfer binary data between two stations short of setting up AMPRnet (aka TCP/IP over AX.25). Most packet data (on VHF and UHF) takes the form of APRS traffic which is digipeated from point to point as as a single packet.

This allows for easy transmission of position reports, and messages, but doesn't provide a good way of sending a datastream from station to station. Part of this is it would be very nice if it was possible to download

Historically, packet communications on Amateur Radio takes the form of AX.25 connections. For Packet BBS, this typically has required a simplex connection between stations, due to the timing and retransmission overhead used in traditional packet operations. 

However, if we design a protocol that can handle file transfer via AX.25 UI frames (aka the same type used by APRS), it can be digipeated from station to station.

# Goals
 * JRT stations should be able to find and each other via APRS position reports
 * File transmissions should be chunked, and then transmitted packet by packet in 1 kilobyte bursts
   - Max packet size of AX.25 is 128 bytes, so this represents 8 packets transmitted per chunk
   - After chunk is transmitting, receiving stations sends signal report, and if any chunks need to be retransmitted
   - Each chunk should be CRC-ed, and compared to ensure proper transmission
 * Awareness of ALOHA
 * Operation via both simplex, and digipetting must be supported, as well as any other AX.25 routing
 * Provide information and directory services - TBD
 * Lay provisions for future extensions -- maybe even a BBS protocol built around the layer.

# ALOHA
From aprs.ogr
```
THE ALOHA CONCEPT: The 1200 baud 144.39 channel can only support an average user load of about 60 to 100 or so average APRS stations or objects in an RF domain based on the typical transmission rates and number of digipeaters and number of hops. . See the breakdown and analysis. This is because any greater load than 100% channel capacity guarantees lost packets due to collisions. . The size of your area that holds the number of users capable of generating 100% channel capacity we call your ALOHA circle. You are responsbile to make sure that your packets get to your surrounding ALOHA network but no further so you do not add QRM to other networks. ALOHA Circle concept.
```

In more direct terms, there is a physical limit of how much data can be transmitted by multiple stations in a given area over RF. By its very nature, RF communications are broadcast, and thus suffer all the same problems one might have if one were to send files to 255.255.255.0

To avoid network congestion, JRT should monitor the frequency band and all AX.25 transmissions on it to determine the current congestion of the local APRS network, and deprioritize transmissions as to not interfere with regular APRS traffic when running on national frequencies.

Currently, the plan is to keep a running average of how many packets are heard and decoded by JRT, and adjust transmission rates as to keep utilization reasonable. A future development goal might be to allow relay of files from JRT node to JRT node as a caching mechanism. 

Theorically, we can allow clients to receive files that they haven't requested by simply monitoring the channel and saving chunks as they receive them. 

APRS runs on the following three principals (from aprs.org)
1. RELIABLE real-time digital communications at the local level
2. A net cycle time of 10 minutes for local (direct/1 hope)
3. A net cycle time of 30 minutes for area wide operations
4. A 1200 baud system operating as a ALOHA random access channel

For handling congestion control, https://www.rfc-editor.org/rfc/rfc2914.txt has more information as do other RFCs.

# File Transmission Operations
## LIST
Lists all files hosted on the JRT node, along with their file ids, and total number of chunks.

## INFO <file_id>
Provides additional information about a given file

## GET_CHUNK <file_id> <chunk_id>
Downloads chunk from a given node. Takes arguments for file name, chunk number


# Transmission Sessions
JRT provides a Layer 3 protocol built over APRS that other applications (including itself) can leverage, such as providing tasks as file transfer and more. In effect, from an application perspective, JRT communications operate as (very slow) TCP/IP connections.

For example: 
 * NODEA requests a file listing from NODEB via APRS msg
 * NODEB returns a transmission id, and a time offset in seconds from when it will start transmitting packets.
 * Max transmission length of a session should be approximately 1 kilobyte in size
   * At 1200-baud, this should represent about 5-10 seconds of transmission time. Rates should be adjusted for 300 and 9600 baud operaiton.
   * Time offset should be calculated by the time it takes for APRS messages to be ACKed
   * Each transmission shall be divided into subchunks with the subchunk size defined as the MTU. Subchunks start at ID 0, and are counted by total length of the data transmitted.
 * NODEA shall wait for the transmission period, and decode as many packets as is possible.
 * If not all packets are decoded, a RESEND request with the specific subchunks shall be sent via APRS message.
  * Transmitting stations shall throttle themselves either based off transmission rates, or observation of other network traffic. JRT traffic shall be considered low priority
 * If further transmissions are needed for all data to be transmitted, NODEA shall transmit that its ready for the next chunk, and the process repeats as above

Furthermore, in certain cases, the station identifer (aka NODEA) may not be the same as the callsign, or, in cases of international operation, the six-character limit of AX.25 callsigns may be a problem -- for example, US hams operating in Canada must legally sign with the suffix /VEx, which can't be encoded in the standard frame.

To solve this problem, transmissions shall provide a mechanism in which callsigns can be transmitting seperately from AX.25 station identiers in band should be necessary.

# Example Session
Assuming two nodes, NODEA, and NODEB running JRT, a basic session look something like this

On startup, and regular intervals (by default 10/30 minutes), a JRT station shall transmit an APRS position report.

JRT capable stations shall be identified by the string JRTxxx, with xx representing a version number

This is represented by standard APRS position reports like so, transmitted as such.

```
NODEA>JRT001,WIDE1-1:Test Station 1
NODEB>JRT001,WIDE1-1:Test Station 2
```

Each JRT client shall keep a list of all nodes it's seen, possible paths to said node if no direct connection is possible, and such.

Assuming NODEA wishs to list all files provided by NODEB, the command structure would look as follows:

## Transmission start
A JRT transmission starts via a three-way handshake via APRS messages.

The initiating station sends a START message, with a JRT command, its MTU, and other relevent data required for packet transmission.

The receiving station will send back its MTU, route, transmission interval, and other information via APRS message to confirm transmission settings, along with a time offset in seconds of when it will transmit packets. The neogtation messages will also include a unique transmission ID, which shall remember the settings negotated, and remain valid for 24 hours.

The initating station shall send a ACK or NAK if it can accept those setings, and then wait for transmission. After transmission, another command can be sent via APRS msgs, or the session can be torn down with a FIN.

Transmitting stations shall send a ACK message back including transmission start time, MTU, and other relevant data.
