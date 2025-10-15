# WhatsApp Calling API for Embedded Sign-up Accounts

## Overview

This API provides a communication bridge between your business telephony system and WhatsApp Business Platform, enabling voice calls to be routed through WhatsApp instead of traditional PSTN networks. The system handles embedded account registration through WhatsApp's Cloud API and manages bidirectional voice call flows.

## Technical Architecture

### Core Components

1. **WhatsApp Business Cloud API Integration**
   - Meta's official WhatsApp Business Platform API
   - Cloud-hosted infrastructure (no on-premise WhatsApp Business API needed)
   - Handles message delivery, call signaling, and account management

2. **SIP/WebRTC Gateway**
   - Converts SIP signaling to WhatsApp-compatible protocols
   - Manages media stream transcoding between traditional telephony and WhatsApp
   - Handles NAT traversal and firewall penetration

3. **API Server**
   - RESTful API endpoints for call initiation and management
   - Webhook receiver for WhatsApp event notifications
   - Authentication and authorization layer
   - Database for user accounts and call logs

4. **Media Server**
   - Real-time media processing (audio encoding/decoding)
   - Supports OPUS codec for WhatsApp compatibility
   - Echo cancellation and noise reduction
   - Call recording functionality

### Protocol Stack

**Primary Protocols:**
- **WebRTC** - For browser-based calling and modern VoIP integration
- **SIP** - For traditional PBX and telephony system integration
- **HTTPS/WSS** - For API communication and real-time signaling
- **OPUS Codec** - Audio codec required by WhatsApp (48kHz, 16-bit)

**WhatsApp Communication:**
- Uses WhatsApp Business API (Cloud API or On-Premises API)
- Message templates for call notifications
- Webhook callbacks for call status updates
- OAuth 2.0 for authentication with Meta

## WebRTC Integration Deep Dive

### What is WebRTC?

WebRTC (Web Real-Time Communication) is an open-source framework that enables real-time voice, video, and data transmission directly between browsers and mobile applications without requiring plugins or third-party software.

### WebRTC Architecture Components

**1. Signaling Layer**
- WebRTC itself doesn't define signaling protocols - you implement your own
- Common approaches: WebSocket (WSS), Socket.IO, or HTTPS long-polling
- Signaling handles session establishment, negotiation, and termination
- Exchanges SDP (Session Description Protocol) offers/answers between peers

**2. ICE (Interactive Connectivity Establishment)**
- Framework for establishing peer-to-peer connections through NATs and firewalls
- Gathers candidate connection paths (host, server reflexive, relayed)
- Tests multiple paths simultaneously to find the best route
- Falls back to relay if direct connection impossible

**3. STUN (Session Traversal Utilities for NAT)**
- Protocol for discovering public IP addresses behind NAT
- Client sends request to STUN server
- Server responds with client's public IP and port
- Required for WebRTC to work through most home/office routers
- Public STUN servers: Google (stun.l.google.com), Twilio, OpenSTUN

**4. TURN (Traversal Using Relays around NAT)**
- Relay server for when direct peer connection fails
- All media traffic routed through TURN server
- Required in ~5-15% of connections (strict NATs, symmetric NATs, corporate firewalls)
- Higher bandwidth cost since all audio flows through server
- Should run on high-bandwidth, low-latency infrastructure

**5. SDP (Session Description Protocol)**
- Text-based format describing multimedia session parameters
- Contains: media types, codecs, IP addresses, ports, bandwidth
- Offer/Answer model: caller creates offer, callee responds with answer
- Renegotiation possible mid-call for changing parameters

**6. RTP/SRTP (Real-time Transport Protocol)**
- Protocol for delivering audio/video over IP networks
- SRTP adds encryption (AES-128/256) for security
- Handles timing, sequencing, and payload type identification
- Works with RTCP for quality monitoring and feedback

### WebRTC Call Flow in Our System

**Phase 1: Connection Initialization**
1. Agent opens web browser interface
2. Browser requests microphone permissions
3. JavaScript creates RTCPeerConnection object
4. Connection configures STUN/TURN servers
5. ICE gathering begins (finds possible connection paths)

**Phase 2: Signaling Exchange**
1. Agent clicks "Call Customer" button
2. Browser creates SDP offer with:
   - Supported codecs (OPUS, PCMU, PCMA)
   - Media capabilities (audio only, bidirectional)
   - ICE candidates (connection paths)
3. Offer sent to signaling server via WebSocket
4. Server forwards to media server
5. Media server generates SDP answer
6. Answer returned through signaling channel
7. Both sides now have connection parameters

**Phase 3: ICE Negotiation**
1. Both peers exchange ICE candidates
2. STUN requests sent to discover public IPs
3. Connectivity checks performed on all candidate pairs
4. Best path selected based on:
   - Latency (prefer lowest)
   - Packet loss
   - Connection type (host > reflexive > relay)
5. If direct connection fails, TURN relay activated

**Phase 4: Media Stream Setup**
1. DTLS handshake for SRTP key exchange
2. Encrypted audio channel established
3. Browser captures microphone audio
4. Audio encoded with OPUS codec
5. RTP packets sent to media server
6. Media server forwards to WhatsApp gateway
7. Return audio path established simultaneously

**Phase 5: Active Call State**
- Bidirectional SRTP streams active
- RTCP reports exchanged every 5 seconds
- Quality metrics monitored (jitter, RTT, packet loss)
- Adaptive jitter buffer manages out-of-order packets
- Automatic gain control (AGC) adjusts volume
- Noise suppression filters background noise
- Echo cancellation prevents feedback loops

**Phase 6: Connection Monitoring**
- ICE connection state: checking → connected → completed
- Signaling state: stable → have-local-offer → stable
- If connection degrades, ICE restart triggered
- If TURN connection fails, call drops with error notification

### WebRTC Configuration Parameters

**RTCConfiguration Object:**
```
{
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { 
      urls: 'turn:your-turn-server.com:3478',
      username: 'user',
      credential: 'pass',
      credentialType: 'password'
    }
  ],
  iceTransportPolicy: 'all',  // or 'relay' to force TURN
  bundlePolicy: 'max-bundle',  // Multiplex on single port
  rtcpMuxPolicy: 'require',    // RTP and RTCP on same port
  iceCandidatePoolSize: 10     // Pre-gather candidates
}
```

**Media Constraints:**
```
{
  audio: {
    echoCancellation: true,
    noiseSuppression: true,
    autoGainControl: true,
    sampleRate: 48000,         // Required for OPUS
    channelCount: 1,           // Mono for voice calls
    volume: 1.0
  },
  video: false                 // Audio-only calls
}
```

### WebRTC Security Features

- **Mandatory Encryption** - All media encrypted with SRTP
- **DTLS** - TLS for key exchange and fingerprint verification
- **Certificate Fingerprinting** - Prevents man-in-the-middle attacks
- **Secure Origins** - WebRTC only works on HTTPS pages (except localhost)
- **Permission Prompts** - Browser requires user consent for microphone access

## SIP Integration Deep Dive

### What is SIP?

SIP (Session Initiation Protocol) is a signaling protocol for initiating, maintaining, and terminating real-time communication sessions including voice calls. It's the industry standard for enterprise telephony and PBX systems.

### SIP Architecture Components

**1. SIP User Agents**
- **UAC (User Agent Client)** - Initiates SIP requests
- **UAS (User Agent Server)** - Responds to SIP requests
- Can be hardware phones, softphones, or PBX systems
- Each agent has unique SIP URI (e.g., sip:agent@domain.com)

**2. SIP Proxy Server**
- Routes SIP requests between user agents
- Handles authentication and authorization
- Performs NAT traversal and topology hiding
- Load balances across multiple servers
- Maintains call state and routing logic

**3. SIP Registrar**
- Database of user locations (IP:port mappings)
- User agents REGISTER their current location
- Registration expires and must be refreshed (typically 3600 seconds)
- Enables mobility - users can move devices

**4. SIP Redirect Server**
- Returns alternative contact addresses
- Doesn't proxy requests, just provides routing information
- Client then contacts destination directly

**5. B2BUA (Back-to-Back User Agent)**
- Acts as both UAC and UAS
- Terminates one SIP dialog and creates another
- Full control over call flow
- Can modify SDP, handle codec transcoding
- Our media server functions as B2BUA

### SIP Message Types

**Requests (Methods):**
- **INVITE** - Initiate a call session
- **ACK** - Acknowledge final response to INVITE
- **BYE** - Terminate an established call
- **CANCEL** - Cancel pending request
- **REGISTER** - Register user agent location
- **OPTIONS** - Query capabilities
- **INFO** - Send mid-call information
- **UPDATE** - Modify session parameters
- **REFER** - Call transfer
- **NOTIFY** - Event notification
- **SUBSCRIBE** - Subscribe to events

**Responses (Status Codes):**
- **1xx** - Provisional (100 Trying, 180 Ringing, 183 Session Progress)
- **2xx** - Success (200 OK)
- **3xx** - Redirection (301 Moved Permanently, 302 Moved Temporarily)
- **4xx** - Client Error (400 Bad Request, 401 Unauthorized, 404 Not Found, 486 Busy Here)
- **5xx** - Server Error (500 Internal Error, 503 Service Unavailable)
- **6xx** - Global Failure (600 Busy Everywhere, 603 Decline)

### SIP Call Flow in Our System

**Outbound Call Scenario (Agent calls Customer via WhatsApp):**

**Step 1: Registration**
```
Agent Phone/Softphone → SIP Proxy
REGISTER sip:proxy.yourdomain.com SIP/2.0
From: <sip:agent@yourdomain.com>
To: <sip:agent@yourdomain.com>
Contact: <sip:agent@192.168.1.100:5060>
Expires: 3600

SIP Proxy → Agent Phone
200 OK
```

**Step 2: Call Initiation**
```
Agent Phone → SIP Proxy
INVITE sip:customer@whatsapp-gateway.yourdomain.com SIP/2.0
From: <sip:agent@yourdomain.com>
To: <sip:customer@whatsapp-gateway.yourdomain.com>
Contact: <sip:agent@192.168.1.100:5060>
Content-Type: application/sdp
[SDP offer with audio codecs, RTP ports]

SIP Proxy → Media Server (B2BUA)
INVITE sip:customer@media-server SIP/2.0
```

**Step 3: Call Progress**
```
Media Server → SIP Proxy
100 Trying

Media Server → SIP Proxy
180 Ringing
[Customer's WhatsApp is ringing]
```

**Step 4: Call Answer**
```
Media Server → SIP Proxy → Agent Phone
200 OK
Content-Type: application/sdp
[SDP answer with accepted codec, media server RTP port]

Agent Phone → SIP Proxy → Media Server
ACK sip:customer@media-server SIP/2.0
```

**Step 5: Media Exchange**
- RTP audio flows: Agent ↔ Media Server ↔ WhatsApp
- Media server transcodes if necessary
- RTCP packets monitor quality

**Step 6: Call Termination**
```
Agent Phone → SIP Proxy → Media Server
BYE sip:customer@media-server SIP/2.0

Media Server → SIP Proxy → Agent Phone
200 OK
```

**Inbound Call Scenario (Customer calls from WhatsApp):**

**Step 1: WhatsApp Call Received**
- Webhook notification received
- API Server identifies customer and available agents
- Determines routing (skill-based, round-robin, priority)

**Step 2: SIP INVITE to Agent**
```
Media Server → SIP Proxy
INVITE sip:agent@yourdomain.com SIP/2.0
From: <sip:whatsapp+1234567890@gateway>
To: <sip:agent@yourdomain.com>
P-Asserted-Identity: "Customer Name" <sip:+1234567890@gateway>
Content-Type: application/sdp
[SDP with OPUS codec from WhatsApp]
```

**Step 3: Agent Phone Alerts**
```
SIP Proxy → Agent Phone
INVITE sip:agent@192.168.1.100:5060 SIP/2.0

Agent Phone → SIP Proxy
100 Trying

Agent Phone → SIP Proxy
180 Ringing
```

**Step 4: Agent Answers**
```
Agent Phone → SIP Proxy → Media Server
200 OK
[SDP answer with agent's codec preferences]

Media Server → Agent Phone
ACK
```

**Step 5: Conversation**
- Media server bridges WhatsApp stream with SIP RTP stream
- Codec transcoding if agent doesn't support OPUS
- Both parties can communicate

**Step 6: Either Party Hangs Up**
```
Agent Phone → Media Server
BYE

Media Server → Agent Phone
200 OK

Media Server terminates WhatsApp call
```

### SIP Header Details

**Essential Headers:**
- **Via** - Path of request, includes protocol, IP, port
- **From** - Initiator of request (includes tag parameter)
- **To** - Intended recipient (tag added in response)
- **Call-ID** - Unique identifier for this call
- **CSeq** - Command sequence number
- **Contact** - Direct reachable address
- **Max-Forwards** - Hop limit (decremented at each proxy)
- **User-Agent** - Software/hardware identification

**Custom Headers for WhatsApp Integration:**
- **X-WhatsApp-ID** - Customer's WhatsApp ID
- **X-Call-Direction** - Inbound or Outbound
- **X-Customer-Name** - Display name from CRM
- **X-Call-Priority** - Routing priority level
- **X-Business-Unit** - Department routing

### SDP in SIP Messages

**Sample SDP Offer:**
```
v=0
o=agent 123456 654321 IN IP4 192.168.1.100
s=Call to WhatsApp Customer
c=IN IP4 192.168.1.100
t=0 0
m=audio 10000 RTP/AVP 111 0 8
a=rtpmap:111 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=fmtp:111 minptime=10; useinbandfec=1
a=ptime:20
a=sendrecv
```

**What this means:**
- Audio media on port 10000
- RTP with AVP profile
- Payload types: 111 (OPUS), 0 (PCMU), 8 (PCMA)
- OPUS at 48kHz stereo with forward error correction
- 20ms packet time
- Bidirectional audio

### SIP Security Mechanisms

**Authentication:**
- **Digest Authentication** - Challenge-response using MD5
- Prevents replay attacks with nonce values
- Password never transmitted in clear text

**Encryption:**
- **TLS (Transport Layer Security)** - Encrypts signaling (SIPS)
- **SRTP (Secure RTP)** - Encrypts media streams
- **SDES** - Session Description for SRTP key exchange
- **DTLS-SRTP** - Alternative key exchange mechanism

**Network Security:**
- **Topology Hiding** - Proxy obscures internal network
- **Session Border Controllers** - Perimeter defense
- **IP Whitelisting** - Only accept from known sources
- **Rate Limiting** - Prevent DoS attacks

### SIP Advantages for This System

✅ **Mature Standard** - 20+ years of development and deployment
✅ **PBX Integration** - Works with existing enterprise systems
✅ **Hardware Support** - Compatible with desk phones, headsets
✅ **Carrier Interoperability** - Can connect to PSTN via SIP trunks
✅ **Feature Rich** - Call transfer, hold, conference built-in
✅ **Debugging Tools** - Wireshark, sngrep, SIPp for testing

### SIP Challenges

❌ **NAT Traversal** - More complex than WebRTC's ICE
❌ **Firewall Configuration** - Multiple ports required (signaling + RTP range)
❌ **Complexity** - Many optional extensions and vendor variations
❌ **Text-Based Protocol** - Higher overhead than binary protocols
❌ **Separate Media/Signaling** - RTP paths need separate management

## WebRTC vs SIP Comparison

| Aspect | WebRTC | SIP |
|--------|--------|-----|
| **Primary Use** | Browser/mobile apps | Enterprise PBX, carriers |
| **Signaling** | Not defined (flexible) | Standardized (RFC 3261) |
| **Media Protocol** | SRTP (encrypted) | RTP or SRTP |
| **NAT Traversal** | ICE/STUN/TURN (built-in) | Requires SBC or ALG |
| **Encryption** | Mandatory | Optional |
| **Codec Negotiation** | SDP via custom signaling | SDP in SIP messages |
| **Installation** | None (browser built-in) | Softphone or hardware |
| **Setup Complexity** | Low | Medium to High |
| **Firewall Friendly** | High | Medium |
| **Call Features** | Basic | Advanced (transfer, hold, etc.) |
| **Debugging** | Difficult | Easier (text protocol) |
| **Best For** | Web apps, mobile, B2C | Contact centers, B2B, PSTN |

## Hybrid WebRTC + SIP Architecture

Our system supports both protocols simultaneously, allowing maximum flexibility:

**Architecture Flow:**
```
[Agent Browser with WebRTC] 
        ↓ WSS Signaling
    [WebRTC Gateway]
        ↓ Internal RTP
    [Media Server / B2BUA]
        ↓ OPUS/RTP
    [WhatsApp Gateway]
        ↓ WhatsApp Protocol
    [Meta Infrastructure]
        ↓
    [Customer WhatsApp App]

SIMULTANEOUSLY:

[Agent SIP Phone]
        ↓ SIP/UDP or SIP/TLS
    [SIP Proxy]
        ↓ SIP
    [Media Server / B2BUA]
        ↓ (same path as above)
```

**Benefits of Hybrid Approach:**
- Agents can use browser OR desk phone
- Call center existing PBX can integrate
- Mobile agents use WebRTC app
- Office agents use SIP phones
- Same media server handles both
- Unified call queuing and routing
- Consistent call quality monitoring

## Required Infrastructure & Resources

### 1. **WhatsApp Business Setup**
- **Meta Business Manager Account** - Required for API access
- **WhatsApp Business Account (WABA)** - Your business identity on WhatsApp
- **Phone Number** - Dedicated phone number registered with WhatsApp Business
- **Business Verification** - Meta verification for official business status
- **Embedded Sign-up Flow** - OAuth integration for customer onboarding

### 2. **Server Infrastructure**
- **Application Server** - Hosts the API logic (minimum 2GB RAM, 2 CPU cores)
- **Media Server** - Handles real-time audio processing (minimum 4GB RAM, 4 CPU cores)
- **Database Server** - PostgreSQL/MySQL for user data and call logs
- **Redis/Cache Layer** - For session management and rate limiting
- **Load Balancer** - For high-availability deployments

### 3. **Network Requirements**
- **Public Static IP Address** - For webhook callbacks from WhatsApp
- **SSL/TLS Certificate** - HTTPS required for all webhook endpoints
- **Open Ports:**
  - 443 (HTTPS) - API and webhook traffic
  - 5060-5061 (SIP) - If using SIP integration
  - 10000-20000 (RTP/UDP) - Media streams
- **Bandwidth** - Minimum 100 Kbps per concurrent call (both upload/download)

### 4. **Third-Party Services**
- **Meta Developer Platform Access**
  - App ID and App Secret
  - System User Access Token (permanent token recommended)
  - Webhook verification token
- **SIP Provider (Optional)** - If integrating with traditional phone systems
  - SIP trunk credentials
  - DID numbers for incoming calls
- **TURN/STUN Servers** - For WebRTC NAT traversal
  - Can use public STUN servers (Google, Twilio)
  - Private TURN server recommended for production

### 5. **Development Resources**
- **API Keys & Credentials**
  - WhatsApp Business Account ID
  - Phone Number ID
  - Business Manager ID
  - Permanent Access Token
- **Webhook URL** - Publicly accessible HTTPS endpoint
- **Development Environment** - Staging environment for testing

## Call Flow Architecture

### Scenario 1: Outgoing Call (You Call Customer)

**Initial Setup Phase:**
1. Customer completes embedded sign-up form on your website/app
2. Form captures customer's WhatsApp phone number
3. API sends embedded sign-up request to WhatsApp Business Platform
4. WhatsApp sends opt-in message to customer
5. Customer accepts and account is linked to your system
6. Customer record stored in database with WhatsApp ID

**Call Initiation Flow:**
1. **Trigger Event**
   - Agent clicks "Call" button in your CRM/Dashboard
   - Automated system triggers call based on business logic
   - API receives call request with customer WhatsApp ID

2. **Pre-Call Notification**
   - API sends message template to customer via WhatsApp Cloud API
   - Message notifies customer of incoming call (required by WhatsApp policy)
   - Template includes business name and call reason
   - Customer receives notification in WhatsApp

3. **SIP/WebRTC Session Setup**
   - API initiates SIP INVITE or WebRTC offer to media server
   - Media server allocates RTP ports and prepares audio processing
   - STUN/TURN servers assist with NAT traversal
   - Session Description Protocol (SDP) negotiated

4. **WhatsApp Call Initiation**
   - API sends call request to WhatsApp Cloud API
   - WhatsApp platform signals customer's device
   - Customer's phone rings in WhatsApp application
   - Call state: "ringing" - webhook notification sent to your server

5. **Call Connection**
   - Customer answers call in WhatsApp
   - WhatsApp establishes media stream to Meta's infrastructure
   - Your media server connects to WhatsApp's media gateway
   - Audio bridges established: Agent ↔ Media Server ↔ WhatsApp ↔ Customer
   - Call state: "connected" - webhook notification sent

6. **Active Call State**
   - Bidirectional audio streams using OPUS codec (48kHz)
   - Media server transcodes if agent uses different codec
   - Echo cancellation and noise reduction applied
   - Optional call recording to database/storage
   - Real-time quality monitoring (jitter, packet loss, latency)

7. **Call Termination**
   - Either party hangs up
   - SIP BYE or WebRTC close signal sent
   - Media streams released
   - Call state: "completed" - final webhook notification
   - Call duration and metadata logged to database

### Scenario 2: Incoming Call (Customer Calls You)

**Prerequisites:**
1. Customer must have previously signed up through embedded flow
2. Customer must have accepted your business terms
3. Your WhatsApp Business number must be saved in customer's contacts (recommended)

**Call Reception Flow:**

1. **Call Initiation by Customer**
   - Customer opens WhatsApp and initiates call to your business number
   - WhatsApp platform receives call request
   - Meta's infrastructure validates business account status

2. **Webhook Notification**
   - WhatsApp Cloud API sends webhook POST to your server
   - Webhook includes: customer WhatsApp ID, timestamp, call type
   - Your API validates webhook signature (security requirement)
   - API queries database to identify customer account

3. **Call Routing Decision**
   - System checks customer priority/segment
   - Determines available agent or queue
   - Applies business hours and routing rules
   - If no agents available: queue or voicemail logic

4. **Agent Notification**
   - API sends alert to assigned agent's interface
   - Displays customer information (CRM data, history)
   - Agent has option to accept or reject
   - Timeout counter starts (e.g., 30 seconds)

5. **Call Acceptance**
   - Agent clicks "Accept" in interface
   - API sends acceptance signal to WhatsApp Cloud API
   - SIP/WebRTC session initiated to agent's device
   - Media server bridges WhatsApp stream with agent connection

6. **Media Stream Establishment**
   - WhatsApp maintains encrypted audio stream with customer
   - Media server receives stream from WhatsApp gateway
   - Agent's audio routed through SIP client or WebRTC browser
   - Full duplex communication established
   - Audio path: Customer ↔ WhatsApp ↔ Media Server ↔ Agent

7. **Active Call Management**
   - Both parties connected and communicating
   - Media server handles codec conversion if needed
   - Call controls available: mute, hold, transfer, record
   - Real-time analytics tracked (duration, quality metrics)
   - CRM automatically logs call details

8. **Call Completion**
   - Either party ends the call
   - WhatsApp sends call ended webhook
   - Media server releases resources
   - Call summary saved: duration, recording link, notes
   - Post-call actions triggered (follow-up tasks, surveys)

### Call Rejection/Timeout Handling

**If agent doesn't answer incoming call:**
1. After timeout period (configurable, e.g., 30s)
2. API sends rejection to WhatsApp
3. Customer hears automated message (if configured)
4. Call logged as "missed"
5. Alternative actions:
   - Route to next available agent
   - Send to voicemail/callback request
   - Queue customer for callback
   - Send automated WhatsApp message with alternatives

## Media Processing Pipeline

### Audio Flow Details

**Outgoing Audio (Agent → Customer):**
1. Agent speaks into microphone
2. Audio captured by SIP client or WebRTC browser
3. Encoded with configured codec (G.711, G.722, OPUS, etc.)
4. Transmitted to media server via RTP or WebRTC data channel
5. Media server decodes audio
6. Re-encodes to OPUS 48kHz (WhatsApp requirement)
7. Sent to WhatsApp media gateway
8. WhatsApp delivers to customer's device
9. Customer hears audio in WhatsApp call

**Incoming Audio (Customer → Agent):**
1. Customer speaks into phone
2. WhatsApp captures and encodes audio (OPUS)
3. Transmitted to Meta's infrastructure (encrypted end-to-end)
4. Delivered to your media server endpoint
5. Media server receives OPUS stream
6. Transcodes to agent's codec if necessary
7. Applies audio enhancements (noise reduction, AGC)
8. Sends to agent via SIP RTP or WebRTC
9. Agent hears customer audio

### Latency Considerations
- **Typical end-to-end latency:** 150-300ms
- **Acceptable maximum:** 400ms
- **Factors affecting latency:**
  - Geographic distance between customer and data center
  - Internet connection quality (both sides)
  - Media server processing overhead
  - WhatsApp gateway routing
  - Network congestion

## Webhook Event System

### Event Types and Handling

**Call Status Events:**
- `call.initiated` - Call request sent to customer
- `call.ringing` - Customer's device is ringing
- `call.answered` - Customer answered the call
- `call.declined` - Customer rejected the call
- `call.busy` - Customer's line is busy
- `call.no_answer` - Customer didn't answer (timeout)
- `call.completed` - Call ended normally
- `call.failed` - Technical failure occurred

**Account Events:**
- `account.registered` - New customer completed embedded sign-up
- `account.verified` - Customer verified phone number
- `account.updated` - Customer information changed
- `account.deleted` - Customer removed their account

**Each webhook payload includes:**
- Event type and timestamp
- Customer WhatsApp ID
- Call ID (for call events)
- Additional metadata (duration, reason codes, etc.)
- Signature for verification

## Security & Authentication

### API Authentication
- **OAuth 2.0** with Meta for WhatsApp Cloud API access
- **JWT tokens** for your API endpoints
- **API key authentication** for server-to-server communication
- **Webhook signature verification** using HMAC-SHA256

### Data Security
- **TLS 1.3** for all API communications
- **End-to-end encryption** maintained by WhatsApp for call audio
- **PII data encryption** at rest in database
- **Token rotation** and expiration policies
- **Rate limiting** to prevent abuse

### Compliance Requirements
- **GDPR** compliance for EU customers
- **CCPA** compliance for California residents
- **WhatsApp Business Policy** adherence
- **Meta Platform Terms** compliance
- **Telecommunications regulations** by jurisdiction

## System Requirements & Specifications

### Performance Metrics
- **API Response Time:** < 200ms (99th percentile)
- **Webhook Processing:** < 100ms
- **Call Setup Time:** < 3 seconds from initiation to ringing
- **Audio Quality:** MOS score > 4.0 (out of 5.0)
- **Concurrent Calls:** Configurable (50-10,000+ based on infrastructure)

### Scalability
- **Horizontal scaling** for API servers behind load balancer
- **Media server clustering** for high call volumes
- **Database sharding** for large user bases
- **CDN integration** for global webhook delivery
- **Auto-scaling** based on call volume metrics

### Monitoring & Logging
- **Real-time call quality metrics** (latency, jitter, packet loss)
- **API request/response logging** with retention policies
- **Error tracking** and alerting system
- **Call detail records (CDR)** for billing and analytics
- **Performance dashboards** for system health monitoring

## Database Schema Overview

### Key Data Entities

**Users/Accounts Table:**
- Account ID, WhatsApp ID, phone number
- Business profile information
- Authentication credentials
- Account status and permissions
- Created/updated timestamps

**Calls Table:**
- Call ID, direction (inbound/outbound)
- Caller and callee WhatsApp IDs
- Call status and state history
- Start time, answer time, end time
- Duration, recording URL
- Quality metrics

**Webhooks Table:**
- Event ID, event type
- Received timestamp
- Payload (JSON)
- Processing status
- Retry count

**Templates Table:**
- Template ID, name, content
- WhatsApp approval status
- Language, category
- Usage statistics

## Integration Options

### Option 1: REST API Integration
- **Best for:** Web applications, mobile apps, custom systems
- **Requires:** HTTP client, JSON parsing
- **Complexity:** Low to medium
- **Real-time:** Polling or webhook-based

### Option 2: WebRTC Integration
- **Best for:** Browser-based calling, web applications
- **Requires:** WebRTC-compatible browser, STUN/TURN configuration
- **Complexity:** Medium
- **Real-time:** True real-time bidirectional communication

### Option 3: SIP Trunk Integration
- **Best for:** Existing PBX systems, call centers, enterprise
- **Requires:** SIP-compatible PBX or softphone
- **Complexity:** Medium to high
- **Real-time:** Standard telephony protocols

### Option 4: SDK Integration
- **Best for:** Native mobile apps (iOS/Android)
- **Requires:** Platform-specific SDK installation
- **Complexity:** Medium
- **Real-time:** Native performance with platform APIs

## Configuration & Setup Process

### Phase 1: Meta Platform Setup
1. Create Meta Business Manager account
2. Create WhatsApp Business App in Meta Developer Portal
3. Add WhatsApp product to your app
4. Complete business verification (can take 1-3 business days)
5. Add phone number and complete phone verification
6. Generate permanent system user access token
7. Configure webhook URL and verify token

### Phase 2: Infrastructure Deployment
1. Provision servers (application, media, database)
2. Configure network (firewalls, load balancers, SSL certificates)
3. Deploy media server with OPUS codec support
4. Set up TURN/STUN servers for NAT traversal
5. Configure SIP gateway if integrating with PBX
6. Deploy Redis for session management
7. Set up monitoring and logging infrastructure

### Phase 3: API Configuration
1. Configure database schemas and migrations
2. Set environment variables and secrets
3. Configure WhatsApp Business API credentials
4. Set up webhook endpoints and signature validation
5. Configure message templates in WhatsApp Manager
6. Set up authentication flows (OAuth, JWT)
7. Configure rate limiting and throttling

### Phase 4: Integration & Testing
1. Test embedded sign-up flow
2. Test outgoing call initiation
3. Test incoming call handling
4. Verify webhook event processing
5. Test media quality and codec transcoding
6. Perform load testing with concurrent calls
7. Test failover and error handling scenarios

### Phase 5: Production Launch
1. WhatsApp Business Account goes live
2. Monitor first production calls closely
3. Collect quality metrics and user feedback
4. Tune media server parameters for optimal quality
5. Scale infrastructure based on actual load

## Technical Limitations & Considerations

### WhatsApp Platform Limits
- **Rate Limits:** API calls throttled per business account tier
  - Tier 1: Up to 1,000 messages per day
  - Tier 2: Up to 10,000 messages per day
  - Tier 3: Up to 100,000 messages per day
  - Unlimited tier: Available after review
- **Template Messages:** Required for initiating calls (must be pre-approved)
- **24-Hour Window:** After customer message, free-form messaging allowed
- **Phone Number Limits:** Each business account can have multiple numbers

### Media Requirements
- **Codec:** OPUS 48kHz mandatory for WhatsApp
- **Bandwidth:** Minimum 50 Kbps, recommended 100 Kbps per call
- **Latency:** Keep under 400ms for good call quality
- **Packet Loss:** Must be under 5% for acceptable audio
- **Jitter:** Should be under 30ms

### Embedded Sign-up Constraints
- **Business Verification:** Required before full API access
- **Customer Consent:** Explicit opt-in required before calls
- **Privacy Policy:** Must be displayed during sign-up
- **Data Retention:** Customer data subject to privacy regulations
- **Opt-out Mechanism:** Must provide easy unsubscribe option

### Call Quality Factors
- **Network Quality:** Both customer and business internet connection
- **Device Compatibility:** Customer's phone must support WhatsApp calling
- **Server Location:** Geographic proximity affects latency
- **Time of Day:** Network congestion during peak hours
- **Mobile vs WiFi:** Customer connection type impacts quality

## Troubleshooting & Diagnostics

### Common Issues and Solutions

**Call Not Connecting:**
- Check WhatsApp Business Account status (active/suspended)
- Verify phone number is properly registered
- Confirm customer has WhatsApp installed and updated
- Check webhook is receiving events
- Verify media server has available resources

**Poor Audio Quality:**
- Measure network latency and packet loss
- Check codec configuration (must be OPUS)
- Verify bandwidth availability
- Review media server CPU usage
- Check for NAT/firewall issues blocking RTP

**Webhook Not Receiving Events:**
- Verify webhook URL is publicly accessible
- Check SSL certificate is valid
- Confirm webhook verification token matches
- Review server logs for incoming requests
- Test with Meta's webhook test tool

**Authentication Failures:**
- Verify access token is not expired
- Check token has correct permissions
- Confirm business account ID is correct
- Review Meta platform status for outages
- Regenerate token if necessary

### Diagnostic Tools
- **Network Testing:** traceroute, ping, MTR for latency analysis
- **SIP Debugging:** tcpdump, wireshark for SIP packet analysis
- **WebRTC Stats:** Built-in browser WebRTC statistics
- **Meta Debug Tool:** For webhook and API troubleshooting
- **Load Testing:** Artillery, JMeter for performance testing

## Next Steps for Implementation

1. **Requirement Analysis** - Define your specific use case and call volumes
2. **Architecture Design** - Choose between WebRTC, SIP, or hybrid approach
3. **Meta Account Setup** - Register business and get WhatsApp Business API access
4. **Infrastructure Planning** - Determine server requirements based on expected load
5. **Development Sprint** - Build API integration following the flows documented above
6. **Testing Phase** - Comprehensive testing of all call scenarios
7. **Pilot Launch** - Start with limited users to validate system
8. **Production Deployment** - Full rollout with monitoring and support
9. **Optimization** - Tune performance based on real-world metrics
