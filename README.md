# ==============================================
# Proyecto: Implementación de SD-WAN
# Autor: Juan Andres Muñoz / Juan Fernando Garcia
# Fecha de Publicación: 18/05/2024
# ==============================================
#  _____  _____    _    _   _  _   _ 
# / ____||  __ \  | |  | | | || \ | |
#| (___  | |  | | | |  | | | ||  \| |
# \___ \ | |  | | | |  | | | || . ` |
# ____) || |__| | | |__| |_| || |\  |
#|_____/ |_____/   \____/(_)_/ |_| \_|
#
import tkinter as tk
from tkinter import ttk
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import numpy as np
from netmiko import ConnectHandler
import re
import time

# Define el buffer circular
class CircularBuffer:
    def __init__(self, capacity): #inicia la capacidad del vector circular
        self.capacity = capacity
        self.buffer = np.zeros(capacity)
        self.index = 0

    def add(self, value): #agrega un valor al vector circular
        self.buffer[self.index] = value
        self.index = (self.index + 1) % self.capacity

    def get_buffer(self): #retorna el vector circular
        return self.buffer

# Initialize the circular buffer with capacity for 24 elements
buffer = CircularBuffer(24)

vyos_router = { #utiliza los datos del ssh de forma que se pueda conectar al router
    "device_type": "vyos",
    "host": "10.195.30.11",
    #"host": "192.168.68.121",
    "username": "vyos",
    "password": "vyos5",
    "port": 22,
}
net_connect = ConnectHandler(**vyos_router)

class Interface: #clase que crea la interfaz    
    def __init__(self, root): #inicia la interfaz
        self.root = root
        self.root.title("Interfaz de gestión")

        # Variables de prueba
        self.variable_1 = tk.StringVar()
        self.variable_2 = tk.StringVar()

        # Almacenamiento de datos
        self.data = []
        self.delay_avg = 0
        self.loss = 0
        self.Automatic = False 
        self.enlace = "A"

        # Crear widgets
        self.create_widgets()

    def create_widgets(self):
        # Botones
        self.button_frame = ttk.Frame(self.root) #crea un frame para los botones
        self.button_frame.pack()                 #empaqueta el frame        

        self.button1 = ttk.Button(self.button_frame, text="Visualizacion en vivo de información", command=self.button1_click)#crea un boton con la accion de button1_click
        self.button1.grid(row=0, column=0, padx=5, pady=5)# Ubicar el botón 1 en la primera fila y primera columna

        self.button2 = ttk.Button(self.button_frame, text="Datos almacenados", command=self.button2_click)
        self.button2.grid(row=0, column=1, padx=5, pady=5)

        self.button3 = ttk.Button(self.button_frame, text="Redireccionamiento manual", command=self.button3_click)
        self.button3.grid(row=0, column=3, padx=5, pady=5)

        self.B1 = ttk.Button(self.button_frame, text="A", command=self.B1)
        self.B1.grid(row=1, column=2, padx=5, pady=5)  # Ubicarlo directamente debajo del botón 3
        self.B1['state'] = 'disabled'  # Deshabilitar inicialmente

        self.B2 = ttk.Button(self.button_frame, text="B", command=self.B2)
        self.B2.grid(row=1, column=3, padx=5, pady=5)  # Ubicarlo directamente debajo del botón 3
        self.B2['state'] = 'disabled'  # Deshabilitar inicialmente

        self.B3 = ttk.Button(self.button_frame, text="C", command=self.B3)
        self.B3.grid(row=1, column=4, padx=5, pady=5)  # Ubicarlo directamente debajo del botón 3
        self.B3['state'] = 'disabled'  # Deshabilitar inicialmente

        self.button4 = ttk.Button(self.button_frame, text="Funcionamiento automatico", command=self.button4_click)
        self.button4.grid(row=0, column=5, padx=5, pady=5)

        self.Bauto = ttk.Button(self.button_frame, text="Activar / Detener", command=self.Bauto)
        self.Bauto.grid(row=1, column=5, padx=5, pady=5)  # Ubicarlo directamente debajo del botón 4
        self.Bauto['state'] = 'disabled'  # Deshabilitar inicialmente

        self.button5 = ttk.Button(self.button_frame, text="Cargar configuración", command=self.button5_click)
        self.button5.grid(row=0, column=6, padx=5, pady=5)

        self.exit_button = ttk.Button(self.root, text="Salir", command=self.exit)   # Botón de salida
        self.exit_button.pack(side='right', anchor='se', padx=5, pady=5)   # Empaquetar el botón de salida

        # Visualización en vivo
        self.live_label = ttk.Label(self.root, text="Visualización en vivo:")   # Etiqueta de visualización en vivo
        self.live_label.pack()  # Empaquetar la etiqueta de visualización en vivo

        self.live_data_label = ttk.Label(self.root, text="")    # Etiqueta de visualización en vivo de datos
        self.live_data_label.pack()   # Empaquetar la etiqueta de visualización en vivo de datos

        # Gráfico QoS
        self.qos_label = ttk.Label(self.root, text="Gráfico Delay vs No de pruebas:")   # Etiqueta del gráfico QoS
        self.qos_label.pack()   # Empaquetar la etiqueta del gráfico QoS

        self.qos_canvas = FigureCanvasTkAgg(self.plot_qos(), master=self.root)  # Crear el gráfico QoS
        self.qos_canvas.draw()  # Dibujar el gráfico QoS
        self.qos_canvas.get_tk_widget().pack()  # Empaquetar el gráfico QoS

        # Actualizar visualización en vivo cada segundo
        self.update_live_data()

    def button1_click(self):                        # Visualización en vivo de información
        self.variable_1.set("Botón 1 presionado")   # Acción para el botón 1
        self.data.append("Botón 1")                 # Almacenar datos
        Pping = 0                                   # Contador de pings    
        while Pping < 2:                            # Realizar 2 pings
            self.PING()                             # Realizar ping  
            Pping += 1                              # Aumentar contador de pings   
        self.update_live_data()                     # Actualizar visualización en vivo   
        self.update_qos_plot()                      # Actualizar gráfico QoS

    def button2_click(self):                        # Datos almacenados
        self.variable_1.set("Botón 2 presionado")
        self.data.append("Botón 2")
        self.update_qos_plot()

    def button3_click(self):                        # Redireccionamiento manual
        self.variable_1.set("Botón 3 presionado")
        self.data.append("Botón 3")
        self.B1['state'] = 'normal'                 # Habilitar botón 1
        self.B2['state'] = 'normal'                 # Habilitar botón 2
        self.B3['state'] = 'normal'                 # Habilitar botón 3

    def button4_click(self):                        # Funcionamiento automático
        self.variable_1.set("Botón 4 presionado")
        self.data.append("Botón 4")
        self.Bauto['state'] = 'normal'             # Habilitar botón de activar/desactivar
        self.update_qos_plot()

    def button5_click(self):                        # Cargar configuración
        self.variable_1.set("Botón 5 presionado")
        self.data.append("Botón 5")
        self.configuracion_inicial()                # Cargar configuración inicial

    def ping_24_datos(self):
        config_commands = ['run ping 172.32.10.15 count 10']    # Comando para realizar ping
        output = net_connect.send_config_set(config_commands, exit_config_mode=False)   # Enviar comando
        match = re.search(r'(\d+)% packet loss', output)    # Buscar porcentaje de pérdida de paquetes
        self.loss = int(match.group(1)) if match else 0    # Almacenar porcentaje de pérdida de paquetes
        match = re.search(r'rtt min/avg/max/mdev = ([\d\.]+)/([\d\.]+)/([\d\.]+)/([\d\.]+)', output)    # Buscar delay promedio
        self.delay_avg = float(match.group(2)) if match else 0  # Almacenar delay promedio
        buffer.add(self.delay_avg)  # Agregar delay promedio al buffer
        self.promedio = np.mean(buffer.get_buffer())*2  # Calcular el doble del promedio de delay
        print("Delays in buffer:", buffer.get_buffer()) # Imprimir buffer de delays
        time.sleep(300) # Esperar 5 minutos

    def PING(self):
        config_commands = ['run ping 172.32.15.10 count 12']    # Comando para realizar ping
        output = net_connect.send_config_set(config_commands, exit_config_mode=False)   # Enviar comando
        match = re.search(r'(\d+)% packet loss', output)    # Buscar porcentaje de pérdida de paquetes
        self.loss = int(match.group(1)) if match else 0   # Almacenar porcentaje de pérdida de paquetes
        match = re.search(r'rtt min/avg/max/mdev = ([\d\.]+)/([\d\.]+)/([\d\.]+)/([\d\.]+)', output)    # Buscar delay promedio
        self.delay_avg = float(match.group(2)) if match else 0  # Almacenar delay promedio
        times = re.findall(r"time=(\d+\.\d+) ms", output)   # Buscar tiempos de delay
        times_float = [float(time) for time in times]   # Convertir tiempos a flotantes
        for valor in times_float:   # Agregar tiempos al buffer
            buffer.add(valor)   # Agregar delay promedio al buffer
        print("Delays in buffer:", buffer.get_buffer()) # Imprimir buffer de delays
        self.promedio = np.mean(buffer.get_buffer())*2  # Calcular el doble del promedio de delay

    def auto(self):
        
        if self.promedio < self.delay_avg or self.loss > 30: # Si el promedio es menor al delay promedio o la pérdida de paquetes es mayor al 30%
            print("Redireccionar al enlace b")          
            self.A = self.delay_avg #almacena el delay promedio
            self.redireccion_B() #funcion que cambia al enlace b
            self.PING() #realiza ping
            if self.promedio < self.delay_avg or self.loss > 30:    # Si el promedio es menor al delay promedio o la pérdida de paquetes es mayor al 30%
                print("Redireccionar al enlace c")
                self.B = self.delay_avg #almacena el delay promedio
                self.redireccion_C() #funcion que cambia al enlace c
                self.PING() #realiza ping
                if self.promedio < self.delay_avg or self.loss > 30:    # Si el promedio es menor al delay promedio o la pérdida de paquetes es mayor al 30%
                    print("Revisar todos los enlaces")
                    self.C = self.delay_avg #almacena el delay promedio
                    self.enlace_decition() #funcion que usa el mejor delay
                else:
                    #funcion que usa el enlace C
                    self.redireccion_C()          
                    print("Redireccionar por C")
            else:
                #funcion que usa el enlace b
                self.redireccion_B()
                print("Redireccionar por b")
        else:
            #funcion que usa el enlace a
            self.redireccion_A()
            print("Redireccionar por A")

    def B1(self):       
        self.redireccion_A()    # Redireccionar al enlace A
        self.PING()
        self.B1['state'] = 'disabled'   # Deshabilitar botón 1
        self.B2['state'] = 'disabled'   # Deshabilitar botón 2
        self.B3['state'] = 'disabled'   # Deshabilitar botón 3

    def B2(self):
        self.redireccion_B()    # Redireccionar al enlace B
        self.PING()
        self.B1['state'] = 'disabled'     
        self.B2['state'] = 'disabled'
        self.B3['state'] = 'disabled'

    def B3(self):
        self.redireccion_C()    # Redireccionar al enlace C
        self.PING()
        self.B1['state'] = 'disabled'
        self.B2['state'] = 'disabled'
        self.B3['state'] = 'disabled'
    
    def Bauto(self):
        self.toggle_work()  # Activar/desactivar el sistema

    def toggle_work(self):  # Activar/desactivar el sistema
        if self.Automatic:
            self.stop_work()  # Detener el sistema
        else:
            self.start_work()  # Iniciar el sistema

    def start_work(self):   # Iniciar el sistema
        self.Automatic = True   # Activar el sistema
        print("Iniciando")
        self.root.after(1000, self.work)    # Continuar el loop si 'Automatic' es True

    def stop_work(self):    # Detener el sistema
        self.Automatic = False  # Desactivar el sistema
        print("saliendo")
        self.Bauto['state'] = 'disabled'    # Deshabilitar botón de activar/desactivar

    def work(self): #funcion que realiza el trabajo
        if self.Automatic:  # Si el sistema está activo
            self.PING()     #realiza ping
            self.auto()     #realiza la funcion auto
            self.root.after(1000, self.work)  # Continuar el loop si 'Automatic' es True

    def enlace_decition(self):  #funcion que decide el mejor enlace
        valor_minimo = min(self.A, self.B, self.C)

        if valor_minimo == self.A:  
            self.redireccion_A()            
        elif valor_minimo == self.B:    
            self.redireccion_B()
        elif valor_minimo == self.C:
            self.redireccion_C()

    def configuracion_inicial(self):    # Cargar configuración inicial
        print("entro a configurar")
        config_commands = [
                            'set policy prefix-list FROM-R2 rule 10 action permit',
                            'set policy prefix-list FROM-R2 rule 10 prefix 172.32.0.0/16',
                            'set policy prefix-list VLAN10 rule 10 action permit',
                            'set policy prefix-list VLAN10 rule 10 prefix 192.168.10.0/24',
                            'set policy prefix-list VLAN10 rule 20 action deny',
                            'set policy prefix-list VLAN10 rule 20 prefix 0.0.0.0/0',
                            'set policy prefix-list VLAN20 rule 10 action permit',
                            'set policy prefix-list VLAN20 rule 10 prefix 192.168.20.0/24',
                            'set policy prefix-list VLAN20 rule 20 action deny',
                            'set policy prefix-list VLAN20 rule 20 prefix 0.0.0.0/0',
                            'delete policy prefix-list VLAN10 rule 30',
                            'delete policy prefix-list VLAN20 rule 30',
                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()
        print("Configuró route-map")
        config_commands = [
                            'set policy route-map VLAN10 rule 10 set local-preference 200',  # Valor más alto indica preferencia
                            'set policy route-map VLAN20 rule 10 set local-preference 200',  # Valor más alto indica preferencia
                            'set protocols bgp neighbor 12.0.0.2 address-family ipv4-unicast route-map import VLAN10',
                            'set protocols bgp neighbor 21.0.0.2 address-family ipv4-unicast route-map import VLAN20',

                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()
        print("Terminó de configurar")

    def redireccion_A(self):    #funcion que redirecciona al enlace A
        print("Entro al A")
        config_commands = [
                            'set policy prefix-list VLAN10 rule 10 action permit',
                            'set policy prefix-list VLAN10 rule 10 prefix 192.168.10.0/24',
                            'set policy prefix-list VLAN10 rule 20 action permit',
                            'set policy prefix-list VLAN10 rule 20 prefix 192.168.20.0/24',
                            'set policy prefix-list VLAN10 rule 30 action deny',
                            'set policy prefix-list VLAN10 rule 30 prefix 0.0.0.0/0',
                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()

        config_commands = [
                            'set policy route-map VLAN10 rule 10 set local-preference 500',  # Valor más alto indica preferencia
                            'set policy route-map VLAN20 rule 10 set local-preference 100',  # Valor más alto indica preferencia
                            'set protocols bgp neighbor 12.0.0.2 address-family ipv4-unicast route-map import VLAN10',
                            'set protocols bgp neighbor 21.0.0.2 address-family ipv4-unicast route-map import VLAN20',
                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()
        print("salio de A")
        self.enlace = "A"

    def redireccion_B(self):    #funcion que redirecciona al enlace B
        print("Entro al B")
        config_commands = [
                            'set policy prefix-list VLAN20 rule 10 action permit',
                            'set policy prefix-list VLAN20 rule 10 prefix 192.168.10.0/24',
                            'set policy prefix-list VLAN20 rule 20 action permit',
                            'set policy prefix-list VLAN20 rule 20 prefix 192.168.20.0/24',
                            'set policy prefix-list VLAN20 rule 30 action deny',
                            'set policy prefix-list VLAN20 rule 30 prefix 0.0.0.0/0',
                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()

        config_commands = [
                            'set policy route-map VLAN10 rule 10 set local-preference 100',  # Valor más alto indica preferencia
                            'set policy route-map VLAN20 rule 10 set local-preference 500',  # Valor más alto indica preferencia
                            'set protocols bgp neighbor 12.0.0.2 address-family ipv4-unicast route-map import VLAN10',
                            'set protocols bgp neighbor 21.0.0.2 address-family ipv4-unicast route-map import VLAN20',
                        ]               
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()
        print("salio del B")
        self.enlace = "B"

    def redireccion_C(self):    #funcion que redirecciona al enlace C
        print("Entro al C")
        config_commands = [
                            'set policy route-map VLAN10 rule 10 set local-preference 20',  # Valor más alto indica preferencia
                            'set policy route-map VLAN20 rule 10 set local-preference 20',  # Valor más alto indica preferencia
                            'set protocols bgp neighbor 12.0.0.2 address-family ipv4-unicast route-map import VLAN10',
                            'set protocols bgp neighbor 21.0.0.2 address-family ipv4-unicast route-map import VLAN20',
                        ]
        output = net_connect.send_config_set(config_commands, exit_config_mode=False, delay_factor=5)
        output = net_connect.commit()
        print("salio del C")
        self.enlace = "C"

    def update_live_data(self): # Actualizar visualización en vivo
        self.live_data_label.config(text=f"Average Delay: {self.delay_avg} ms // average Packet loss: {self.loss} % // Envio de información por: Enlace {self.enlace}") # Actualizar etiqueta de visualización en vivo
        self.root.after(1000, self.update_live_data)    # Actualizar cada segundo

    def plot_qos(self):   # Gráfico QoS
        fig, ax = plt.subplots()    # Generar gráfico QoS inicial
        ax.plot(range(24), buffer.get_buffer())   # Graficar buffer de delays
        ax.set_xlabel("No de prueba")  # Etiqueta para el eje X
        ax.set_ylabel("Delay (ms)")  # Etiqueta para el eje Y
        return fig

    def update_qos_plot(self):  # Actualizar gráfico QoS
        self.qos_canvas.get_tk_widget().destroy()   # Eliminar el widget existente
        new_plot = self.plot_qos()  # Generar un nuevo gráfico QoS
        self.qos_canvas = FigureCanvasTkAgg(new_plot, master=self.root) # Crear un nuevo widget
        self.qos_canvas.draw()  # Dibujar el nuevo widget
        self.qos_canvas.get_tk_widget().pack()  # Mostrar el nuevo widget

    def exit(self):
        # Este método cerrará la aplicación y conexión ssh
        net_connect.disconnect()
        self.root.destroy()

if __name__ == "__main__":  # Iniciar la aplicación
    root = tk.Tk()  # Crear la ventana principal
    app = Interface(root)   # Crear la aplicación
    root.mainloop() # Iniciar el loop de la aplicación
