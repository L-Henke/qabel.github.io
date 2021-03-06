---
title: Client Drop
---
# Qabel drop messages
## The structure and encryption of Qabel drop messages, transported using the Qabel Drop protocol.

### Terminology

| Term | Meaning |
| ---- | ------- |
| address | qabel drop address as URL |
| acknowledge id | identifier which is used to acknowledge the recipience of a drop message |
| sender | person or system creating and sending qabel messages |
| recipient | person or system receiving qabel messages |
| client | qabel client (software the sender is using) |
| server | qabel server (drop server referred by the address) |
| message payload | data the sender intents to deliver to the recipient |
| message | json object the client sends to the server |


### Attendees

* Sender
* Server

### Use cases:

#### Send message
Alice (sender) wants to write a message to Bob (recipient). Alice has Bobs address and public key, therefore Bob has a valid inbox.
The client will send the message to the server.

**On sending success:**
The server will return the specific [response](https://github.com/Qabel/intern-doc/wiki/Qabel-Protocol-Drop#methods).
The message will be added to the history if this is wanted.

**On sending failure:**
The server will return the specific [response](https://github.com/Qabel/intern-doc/wiki/Qabel-Protocol-Drop#methods).
The user will be informed and/or the client will try to resend the message.

#### Get message
Alice (receiver) wants to receive new messages. Alice has a private key and an address, therefore a valid inbox.
The client will request new messages from the server.

**On receiving success:**
The server will return a specific [response] (https://github.com/Qabel/intern-doc/wiki/Qabel-Protocol-Drop#methods) including the new messages.

**On receiving failure:**
The server will return a specific [response] (https://github.com/Qabel/intern-doc/wiki/Qabel-Protocol-Drop#methods).
The user will be informed and/or the client will retry to receive the messages.

### Format and structure of a message
A message is packed into JSON containing the following fields:

| Key | Type | Description |
| --- | ---- | ----------- |
| **version** | INT | This key describes the version of the Qabel Drop Message protocol. |
| **time_stamp** | INT | Date of message generation. |
| **acknowledge_id** | STR | Acknowledge ID for acknowledging this message |
| **sender** | STR | The [key id](../Components-Crypto/#key-identifier) of sender's public key. |
| **model_object** | STR | The name of the model object that handles this message. |
| **data** | JSON object | The payload of the message. |

#### Summary

    drop_message    = "{"
                    'version' : INT,
                    'time_stamp' : LONG,
                    'acknowledge_id': STR,
                    'sender' : STR,
                    'model_object' : STR,
                    'data' : { ... }
                    "}"


#### Special messages
Special drop messages are defined for message acknowledgement and error notification.
Special messages have the `model_object` set to `drop`.
Other models MUST NOT use the model object `drop`.

##### Acknowledging
The drop client automatically acknowledges each incoming drop message.
In order to acknowledge message `N`, the client generates a drop messages where
`acknowledge_id` is `N`. The `data` field of an acknowledge message is undefined.
Messages with the `acknowledge_id` set to `0` (zero) are not acknowledged.

##### Error notification
If the drop client caught an error while handling drop message with `acknowledge_id` `N`,
the client generates a drop message to notify the original sender about the error.
The `acknowledge_id` of this notification is `N`.
The `data` field contains error descriptions:

    error           = "{"
                    'type' : STR,
                    'error_object' : { ... }
                    "}"

The following error types:

| Type | Description |
| ---- | ----------- |
| UNKNOWN_MODULE | Given `module_object` is unknown |


### Sequence diagrams

#### Send message:
![Sequence diagram send message](/images/sequencediagram_send_messages.png)

#### Receive messages:
![Sequence diagram receive messages](/images/sequencediagram_receive_messages.png)

### Encryption
Encryption follows the general encrypt-and-sign scheme described in the [general crypto documentation](../Components-Crypto/#encrypt-and-sign).

#### Padding
Prior to encryption, the serialized messages are padded to a fixed size of 2048 byte in order to avoid the metadata content length.

#### Asymmetric encryption.
The padded message is encrypted using AES in CTR mode with a random key of 256 bits and an IV, consisting of a 96 bit random nonce and a 32 bit counter which always starts to count at 1, forming the ciphertext.
The AES key is encrypted with RSA OAEP encryption scheme using the recipients public encryption key (cf. [multiple key-pair concept](../Components-Crypto/#multiple-key-pair-concept)).
The encrypted message is created by concatenating three fields without any delimiter; the encrypted AES key, the AES nonce (first 12 bytes of the IV), and the ciphertext.
For example, with a 2048 bit RSA key the encrypted message looks like this:

```json
  RSA_encrypt([256 byte AES key])[12 bytes AES nonce][2K-1-2048-12 bytes AES ciphertext]
```

#### Signature

A digest of the final encrypted message including header, the encrypted AES key, the AES nonce, and the ciphertext is created via the SHA512 hash function. The digest is signed with the senders private signing key (cf. [multiple key-pair concept](../Components-Crypto/#multiple-key-pair-concept)) using the RSASSA-PSS scheme. The signature is appended to the previously created encrypted message, forming the authenticated encrypted message.

### Transport format
The following section defines the protocl data unit (PDU) of the Qabel Drop protocol.
This PDU is transmitted as body of a HTTP request/response.

The PDU starts with a version byte to indicate the version of PDU format.

#### PDU Version 0

| Message part | Field | Description | Length (in Bytes) |
| ------------ | ----- | ----------- | ---------------: |
| **Header** (unencrypted) | Version | Version of the Qabel Drop PDU format (here 0)| 1 |
| **Key** | Key (encrypted with the public key of the recipient) | Newly generated key used with the symmetric block cipher to encrypt the data | 256 |
|         | Initialisation data (unencrypted) | Data to initialize a symmetric block cipher (e.g. an nonce) | 12 |
| **Data** (encrypted with symmetric block cipher) | Payload + Padding | Original Qabel drop message appended by padding | 2048 |
| **Signature** | Signature | Digital signature of Header, Key and Data made with sender's private key | 256 |

### History / Persistence

The core will handle the drop history but this functionality is not yet defined.
**TODO**: Messages which are needed across all devices of an identity must be shared across all devices. The exact way has to be defined.


