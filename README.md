# raspberrypi-whatsapp

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
67
68
69
70
71
72
73
74
75
76
77
78
79
80
81
82
83
84
85
86
87
88
89
90
91
92
93
94
95
96
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
134
135
136
137
138
139
140
141
142
143
144
145
146
147
148
149
150
151
152
153
154
155
156
157
158
159
160
161
162
163
164
165
166
167
168
169
170
171
172
173
174
175
176
177
178
179
180
181
182
183
184
185
186
187
188
189
190
191
192
193
194
195
196
197
198
199
200
201
202
203
204
205
206
207
208
209
210
211
212
213
214
215
216
217
218
219
#!/usr/bin/env
#!/usr/bin/python
 
import RPi.GPIO as GPIO
import time
import base64 
import os
import urllib
import numpy as np
 
 
from datetime import datetime
from dateutil import tz
from Yowsup.connectionmanager import YowsupConnectionManager
 
def inicia_whatsapp():
    #Se hacen publicas las variables de la conexion
    global y 
    global signalsInterface
    global methodsInterface
    global contador
    global vaciado_mensajes_offline
    y = YowsupConnectionManager() 
    y.setAutoPong(True) #Responder a los ping que hagan
    signalsInterface = y.getSignalsInterface() #Se recibiran las senales
    methodsInterface = y.getMethodsInterface() #Se utilizaran los metodos
 
    signalsInterface.registerListener("message_received", onMessageReceived)
    signalsInterface.registerListener("receipt_messageSent", onMessageSent)
    signalsInterface.registerListener("receipt_messageDelivered", onMessageDelivered)
    signalsInterface.registerListener("auth_success", onAuthSuccess)
    signalsInterface.registerListener("ping", onPing)
    signalsInterface.registerListener("disconnected", onDisconnected)
 
    methodsInterface.call("auth_login", (username, password))
    methodsInterface.call("ready")
     
    #Leer todos los mensajes offline y no analizarlos
    contador = 0
    vaciado_mensajes_offline = False
    while (contador<10): #Contador se pondra a cero cada vez que salta la interrupcion de un mensaje
        contador += 1
        time.sleep(0.1)
    #Cuando sale del while ha pasado 1 segundo sin recibir mensajes
    vaciado_mensajes_offline = True
    methodsInterface.call("message_send", ("34639xxxxxx@s.whatsapp.net", 'Reset'))
 
 
def onDisconnected(reason): #Cuando se desconecta
        print "Closed by:" + reason
    inicia_whatsapp() #Si se desconecta vuelve a iniciarse automaticamente
 
def onAuthSuccess(username): #Cuando la autentificacion es correcta
        print "Logged in with:" + username
 
def onMessageSent(jid, messageId): #Cuando se envia un mensaje
        print "Message was sent successfully to %s" % jid
 
def onMessageDelivered(jid, messageId): #Cuando se verifica que el mensaje ha sido leido
        print "Message was delivered successfully to %s" %jid
        methodsInterface.call("delivered_ack", (jid, messageId))
 
def onMessageReceived(messageId, jid, messageContent, timestamp, wantsReceipt, pushName, isBroadCast): #Cuando se recibe un mensaje
    global contador
    contador = 0
    methodsInterface.call("message_ack", (jid, messageId))
    if (vaciado_mensajes_offline):  
        AnalizaMensajes(jid,messageContent) 
    print "Mensaje: " + messageContent + " Usuario:" + jid + " MensajeId:" + messageId
 
def onPing(pingId): #Cuando se recibe un ping
        methodsInterface.call("pong", (pingId,))
 
def AnalizaMensajes(numero,mensaje):
    global consigna
    mensaje_por_defecto = 'Acciones validas:' + "\n"
    mensaje_por_defecto += 'ver -> ver precios' + "\n"
    mensaje_por_defecto += 'cargar -> cargar precio nuevos' + "\n"
    mensaje_por_defecto += 'consigna -> Asignar consigna' + "\n"
 
    mensaje = mensaje.lower() #convertir el mensaje siempre a minusculas
 
    if numero[:11] in telefonos:
        if mensaje == 'ver':
            enviar_precios(numero)
        elif mensaje == 'cargar':
            carga_precios(numero)
        elif mensaje[:len('consigna')] == 'consigna':
            consigna = float(mensaje.strip('consigna '))
            methodsInterface.call("message_send", (numero, "Nueva consiga: " + str(consigna)))
        else:
            methodsInterface.call("message_send", (numero, mensaje_por_defecto))
    else:
        if mensaje == 'password 123456':
            methodsInterface.call("message_send", (numero, "Password correcto"))
            telefonos.append(numero[:11])
            archivo = open('telefonos.txt','a')
            archivo.write(numero[:11]+'\n')
            archivo.close()
        else:
            methodsInterface.call("message_send", (numero, "Envie un mensaje con la palabra password seguido del numero de validacion\nEjemplo:\npassword 000000"))
 
def enviar_precios(numero):
    mensaje = 'Precios dia: ' + datetime.now(tz.gettz('Europe/Madrid')).strftime("%Y-%m-%d") + '\n'
    for i in range(24):
        mensaje += 'Hora '+str(i)+': '+str(precio_hora[i])+' \n'
    mensaje += 'Consigna: ' + str(consigna)+ '\n'
    mensaje += 'Modo Ahorro: ' + str(modo_ahorro)+ '\n'
    methodsInterface.call("message_send", (numero, mensaje))
 
def carga_precios(numero):
    #################################################################################
    #                                       #
    # En la variable media se tiene el precio medio                 #
    # En la variable precio_hora[i] se tiene el precio de la hora i         #
    #                                       #
    #################################################################################
    global tarifa
    global media
    global precio_hora
     
    tarifa = datetime.now(tz.gettz('Europe/Madrid')).strftime("%Y%m%d") #tarifa es la fecha actual en formato 20140418 ano mes dia
    os.system('rm *.csv*') #Eliminar archivos anteriores
    urllib.urlretrieve('http://www.esios.ree.es/Solicitar/PVPC_DD_' + tarifa + '.csv&es ','PVPC_DD_' + tarifa + '.csv') #Descargar archivo de REE
    os.system('unzip -o PVPC_DD_'+ tarifa +'.csv') #Decomprimir
 
 
    texto = 'Precio Tarifa General'
    archivo = open('/home/pi/yowsup/src/PVPC_DD_'+tarifa+'.csv','r')
 
    #Lee la primera entrada con el valor medio (no lo usamos)
    linea=archivo.readline()
    while (linea[:len(texto)] <> texto):
        linea=archivo.readline()
 
    linea = linea.strip(texto+';') #Elimina la primera parte del texto
    linea = linea[:-2] #Elimina fin de linea y retorno de carro
    linea = linea.replace(',','.') #Remplaza comas por puntos
    linea = linea.split(';')  #Crea un array con los valores separados por ;
    media = float (linea[0])
 
    #Lee la segunda entrada con los valores horarios
    linea=archivo.readline()
    while (linea[:len(texto)] <> texto):
        linea=archivo.readline()
 
    linea = linea.strip(texto+';') #Elimina la primera parte del texto
    linea = linea[:-2] #Elimina fin de linea y retorno de carro
    linea = linea.replace(',','.') #Remplaza comas por puntos
    linea = linea.split(';') #Crea un array con los valores separados por ;
 
    precio_hora = np.empty([24],dtype=np.float)
    for i in range(24):
        precio_hora[i] = float(linea[i])
 
 
    archivo.close()
    enviar_precios(numero)
 
#Parametros de Configuracion de WhatsApp
password = "1Ohumn1JoqwTVOec5K0n9GHrCus="
password = base64.b64decode(bytes(password.encode('utf-8')))
username = '34650xxxx'
 
#Configuracion de los puertos GPIO
GPIO.setmode(GPIO.BCM)
GPIO.setup(24, GPIO.IN, pull_up_down = GPIO.PUD_UP)
GPIO.setup(25, GPIO.IN, pull_up_down = GPIO.PUD_UP)
GPIO.setup(17, GPIO.OUT)
GPIO.output(17,0)
 
#Archivo de texto con los telefono validos
telefonos = [] #Se crea una lista con los telefonos validos
archivo = open('/home/pi/yowsup/src/telefonos.txt','r')
linea=archivo.readline()
while linea!="":
     telefonos.append(linea[:11])
     linea=archivo.readline()
archivo.close()
 
inicia_whatsapp()
 
consigna = 100.0 #precio MWh maximo
modo_ahorro = False
carga_precios('34639xxxxxx@s.whatsapp.net')
 
while True:
    time.sleep(1)
     
    if not (tarifa == datetime.now(tz.gettz('Europe/Madrid')).strftime("%Y%m%d")): #cambia el dia
        carga_precios('34639xxxxxx@s.whatsapp.net')
 
    if (precio_hora[datetime.now(tz.gettz('Europe/Madrid')).hour]>consigna):
        if (not modo_ahorro):
            methodsInterface.call("message_send", ("34639xxxxxx@s.whatsapp.net", 'Modo ahorro activado'))
            GPIO.output(17,1)
        modo_ahorro = True
    else:
        if (modo_ahorro):
            methodsInterface.call("message_send", ("34639xxxxxx@s.whatsapp.net", 'Modo ahorro desactivado'))
            GPIO.output(17,0)
        modo_ahorro = False
     
 
    if(GPIO.input(24) ==0):
        print('Button 1 on')
        while(GPIO.input(24) ==0):
            time.sleep(0.5) 
        print('Button 1 off')
 
 
    if(GPIO.input(25) == 0):
        print('Button 2 on')
        while(GPIO.input(25) == 0):
            time.sleep(0.5) 
        print('Button 2 off')
 
         
GPIO.cleanup()
