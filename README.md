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
 * Operation via both simplex, and digipetting must be supported, as well as any other AX.25 routing
 * Provide information and directory services - TBD
 * Lay provisions for future extensions -- maybe even a BBS protocol built around the layer.

 # 
