#Proyecto_Fundamentos
"""
Teclado Estenógrafo en Español para Raspberry Pi Pico W

Este código implementa un teclado estenógrafo que permite escribir en español
usando combinaciones de teclas simultáneas para formar palabras completas.

Funciones especiales:
- GP7: Modo numérico (toggle)
- GP10: Mayúscula (momentáneo)
- GP16: Espacio
- GP19: Borrar

"""

import board
import digitalio
import usb_hid
from adafruit_hid.keyboard import Keyboard
from adafruit_hid.keycode import Keycode
import time

#Configuración del teclado HID
kbd = Keyboard(usb_hid.devices)

#Mapeo de pines GPIO a letras del teclado estenógrafo
#Según la imagen proporcionada
PIN_MAP = {
    board.GP1: 'M',   # Fila superior izquierda
    board.GP2: 'S',
    board.GP3: 'T',
    board.GP4: 'D',   # Segunda fila izquierda
    board.GP5: 'Q',
    board.GP6: 'J',
    board.GP7: '^',   # Función especial: modo numérico
    board.GP8: 'E',   # Tercera fila izquierda
    board.GP9: 'P',
    board.GP10: 'SHIFT',  # Función especial: mayúscula
    board.GP11: 'A',  # Fila inferior
    board.GP12: 'O',
    board.GP13: 'E',
    board.GP14: 'L',  # Lado derecho
    board.GP15: 'B',
    board.GP16: 'SPACE',  # Función especial: espacio
    board.GP17: 'C',
    board.GP18: 'N',
    board.GP19: 'BACKSPACE',  # Función especial: borrar
    board.GP20: 'R',
    board.GP21: 'I',
    board.GP26: 'G',
    board.GP27: 'F',
    board.GP28: 'U'
}

#Mapeo para modo numérico (GP1-GP12 excepto GP7 y GP10)
NUMERIC_MAP = {
    board.GP1: '1',   # M -> 1
    board.GP2: '2',   # S -> 2
    board.GP3: '3',   # T -> 3
    board.GP4: '4',   # D -> 4
    board.GP5: '5',   # Q -> 5
    board.GP6: '6',   # J -> 6
    board.GP8: '7',   # E -> 7
    board.GP9: '8',   # P -> 8
    board.GP11: '9',  # A -> 9
    board.GP12: '0'   # O -> 0
}

class StenographKeyboard:
    def __init__(self):
        """
        Inicializa el teclado estenógrafo configurando todos los pines
        y estableciendo los estados iniciales.
        """
        self.pins = {}
        self.previous_state = {}
        self.current_chord = set()  # Conjunto de teclas presionadas simultáneamente
        self.numeric_mode = False   # Estado del modo numérico
        self.shift_pressed = False  # Estado de mayúscula
        
        # Tiempos generosos para formación de combinaciones - como aprender piano
        self.chord_formation_time = 2.0   # Tiempo total para formar una combinación (2 segundos)
        self.chord_stability_time = 0.6   # Tiempo de espera después de última tecla (600ms)
        self.last_press_time = 0          # Momento de la última tecla presionada
        self.first_press_time = 0         # Momento de la primera tecla presionada
        self.chord_in_progress = False    # Indica si estamos formando una combinación
        
        # Estados para debugging - ayuda a entender qué está pasando
        self.debug_mode = True
        
        # Configurar todos los pines como entrada con pull-down
        for pin in PIN_MAP.keys():
            gpio_pin = digitalio.DigitalInOut(pin)
            gpio_pin.direction = digitalio.Direction.INPUT
            gpio_pin.pull = digitalio.Pull.DOWN  # Pull-down porque los botones van a 3V
            self.pins[pin] = gpio_pin
            self.previous_state[pin] = False
        
        print("Teclado estenógrafo iniciado")
        print("Modo normal activado")
    
    def read_pins(self):
        """
        Lee el estado actual de todos los pines y detecta cambios.
        
        Esta función es como el "oído" del teclado - constantemente escucha
        qué teclas están siendo presionadas o liberadas.
        
        Retorna:
            changes (bool): True si hay algún cambio en el estado de las teclas
            current_state (dict): Estado actual de todos los pines
            pressed_pins (set): Conjunto de pines que están presionados ahora
            released_pins (set): Conjunto de pines que se acaban de liberar
        """
        current_state = {}
        pressed_pins = set()
        released_pins = set()
        changes = False
        
        # Leer el estado de cada pin
        for pin, gpio_pin in self.pins.items():
            current_state[pin] = gpio_pin.value
            
            # Detectar teclas recién presionadas
            if current_state[pin] and not self.previous_state[pin]:
                pressed_pins.add(pin)
                changes = True
                if self.debug_mode:
                    letter = PIN_MAP.get(pin, "ESPECIAL")
                    print(f"✓ Tecla presionada: {letter}")
            
            # Detectar teclas recién liberadas
            elif not current_state[pin] and self.previous_state[pin]:
                released_pins.add(pin)
                changes = True
                if self.debug_mode:
                    letter = PIN_MAP.get(pin, "ESPECIAL")
                    print(f"✗ Tecla liberada: {letter}")
        
        self.previous_state = current_state.copy()
        return changes, current_state, pressed_pins, released_pins
    
    def handle_special_keys(self, pressed_pins):
        """
        Maneja las teclas con funciones especiales antes del procesamiento normal.
        
        Las teclas especiales son como "comandos mágicos" que tienen efectos inmediatos
        sin necesidad de formar combinaciones complejas.
        
        Args:
            pressed_pins (set): Conjunto de pines que están presionados
            
        Returns:
            bool: True si se procesó una tecla especial, False en caso contrario
        """
        # Verificar que solo hay una tecla presionada para funciones especiales
        if len(pressed_pins) != 1:
            return False
        
        single_pin = list(pressed_pins)[0]
        
        # Modo numérico (GP7) - Como cambiar el "idioma" del teclado
        if single_pin == board.GP7:
            self.numeric_mode = not self.numeric_mode
            mode_text = "numérico" if self.numeric_mode else "estenográfico"
            print(f"🔄 Modo {mode_text} activado")
            return True
        
        # Espacio (GP16) - Función simple y directa
        if single_pin == board.GP16:
            kbd.send(Keycode.SPACE)
            if self.debug_mode:
                print("⎵ Espacio enviado")
            return True
        
        # Borrar (GP19) - Como el botón de borrar de cualquier teclado
        if single_pin == board.GP19:
            kbd.send(Keycode.BACKSPACE)
            if self.debug_mode:
                print("⌫ Borrar enviado")
            return True
        
        return False
    
    def process_chord(self, pressed_pins):
        """
        Procesa una combinación de teclas (chord) y la convierte en texto.
        """
        if not pressed_pins:
            return
        
        # Filtrar teclas especiales del chord
        chord_pins = {pin for pin in pressed_pins 
                     if pin not in [board.GP7, board.GP16, board.GP19]}
        
        if not chord_pins:
            return
        
        # Modo numérico
        if self.numeric_mode:
            self.process_numeric_input(chord_pins)
            return
        
        # Modo estenográfico normal
        self.process_stenographic_input(chord_pins)
    
    def process_numeric_input(self, pressed_pins):
        """
        Procesa la entrada en modo numérico.
        """
        numbers = []
        for pin in pressed_pins:
            if pin in NUMERIC_MAP and pin != board.GP10:  # Excluir shift en modo numérico
                numbers.append(NUMERIC_MAP[pin])
        
        if numbers:
            # Ordenar los números según el orden físico de las teclas
            pin_order = [board.GP1, board.GP2, board.GP3, board.GP4, board.GP5, 
                        board.GP6, board.GP8, board.GP9, board.GP11, board.GP12]
            sorted_numbers = []
            for pin in pin_order:
                if pin in pressed_pins and pin in NUMERIC_MAP:
                    sorted_numbers.append(NUMERIC_MAP[pin])
            
            number_string = ''.join(sorted_numbers)
            self.type_text(number_string)
            print(f"Número enviado: {number_string}")
    
    def process_chord(self, pressed_pins):
        """
        Procesa una combinación de teclas (chord) y la convierte en texto.
        
        Esta función es como un "traductor" que toma tu combinación de teclas
        y la convierte en la palabra correspondiente. Es el corazón del sistema
        estenográfico.
        
        Args:
            pressed_pins (set): Conjunto de pines que formaron la combinación
        """
        if not pressed_pins:
            return
        
        # Verificar si shift (GP10) está incluido en la combinación
        if board.GP10 in pressed_pins:
            self.shift_pressed = True
            # Remover shift del chord para el procesamiento normal
            pressed_pins = pressed_pins - {board.GP10}
        
        # Filtrar teclas especiales del chord (excepto shift que ya manejamos)
        chord_pins = {pin for pin in pressed_pins 
                     if pin not in [board.GP7, board.GP16, board.GP19]}
        
        if not chord_pins:
            return
        
        # Decidir si procesamos en modo numérico o estenográfico
        if self.numeric_mode:
            self.process_numeric_input(chord_pins)
        else:
            self.process_stenographic_input(chord_pins)
    
    def process_stenographic_input(self, pressed_pins):
        """
        Procesa la entrada estenográfica convirtiendo combinaciones en palabras.
        
        Esta función es como un diccionario que busca qué palabra corresponde
        a tu combinación de letras. Si no encuentra una palabra exacta,
        escribirá las letras individuales como respaldo.
        
        Args:
            pressed_pins (set): Conjunto de pines que forman la combinación
        """
        # Crear la combinación de letras presionadas
        chord_letters = set()
        for pin in pressed_pins:
            if pin in PIN_MAP:
                chord_letters.add(PIN_MAP[pin])
        
        if not chord_letters:
            return
        
        # Crear clave para buscar en el diccionario (letras ordenadas alfabéticamente)
        chord_key = ''.join(sorted(chord_letters))
        
        if self.debug_mode:
            print(f"🔍 Buscando combinación: '{chord_key}'")
        
        # Buscar en el diccionario estenográfico
        if chord_key in STENOGRAPH_DICT:
            word = STENOGRAPH_DICT[chord_key]
            
            # Aplicar mayúscula si shift está presionado
            if self.shift_pressed:
                word = word.capitalize()
                if self.debug_mode:
                    print("🔠 Aplicando mayúscula")
            
            self.type_text(word)
            print(f"📝 '{chord_key}' → '{word}'")
            
        else:
            # Si no encuentra la combinación, escribir las letras individuales
            # Esto actúa como "red de seguridad" para no perder tu escritura
            letters = ''.join(sorted(chord_letters))
            if self.shift_pressed:
                letters = letters.upper()
            
            self.type_text(letters)
            if self.debug_mode:
                print(f"⚠️  Combinación no encontrada. Escribiendo letras: '{letters}'")
                print("💡 Consejo: Puedes agregar esta combinación al diccionario")
        
        # Resetear shift después de usar
        self.shift_pressed = False
    
    def type_text(self, text):
        """
        Envía texto al computador usando HID.
        """
        for char in text:
            if char.isalpha():
                # Convertir letra a keycode
                if char.isupper():
                    kbd.press(Keycode.SHIFT)
                    kbd.press(getattr(Keycode, char.upper()))
                    kbd.release(getattr(Keycode, char.upper()))
                    kbd.release(Keycode.SHIFT)
                else:
                    kbd.press(getattr(Keycode, char.upper()))
                    kbd.release(getattr(Keycode, char.upper()))
            elif char.isdigit():
                # Números
                if char == '0':
                    kbd.press(Keycode.ZERO)
                    kbd.release(Keycode.ZERO)
                else:
                    keycode = getattr(Keycode, f"{'ONE TWO THREE FOUR FIVE SIX SEVEN EIGHT NINE'.split()[int(char)-1]}")
                    kbd.press(keycode)
                    kbd.release(keycode)
            # Añadir espacio después de cada palabra
            time.sleep(0.01)
    
    def run(self):
        """
        Bucle principal del teclado estenógrafo con lógica mejorada de temporización.
        
        Esta función es como el "director de orquesta" que coordina todo el proceso.
        Implementa un sistema inteligente de tres fases:
        
        1. ESCUCHA: Detecta cuando empiezas a presionar teclas
        2. ACUMULACIÓN: Recolecta todas las teclas que presiones durante un tiempo generoso  
        3. PROCESAMIENTO: Cuando terminas, convierte la combinación en una palabra
        
        Piensa en esto como escribir una palabra en el aire con las manos - necesitas
        tiempo para "dibujar" toda la palabra antes de que alguien pueda leerla.
        """
        print("🎹 Teclado estenógrafo iniciado con temporización mejorada")
        print("💡 Presiona combinaciones de teclas y mantén hasta completar la palabra")
        print("📝 El sistema esperará a que termines antes de escribir")
        
        while True:
            current_time = time.monotonic()
            changes, current_state, newly_pressed, newly_released = self.read_pins()
            
            # Obtener todas las teclas actualmente presionadas
            currently_pressed = {pin for pin, state in current_state.items() if state}
            
            if changes:
                # ==================== FASE 1: DETECCIÓN DE NUEVAS TECLAS ====================
                if newly_pressed:
                    # Manejar teclas especiales inmediatamente (no necesitan combinaciones)
                    if len(currently_pressed) == 1 and self.handle_special_keys(newly_pressed):
                        # Limpiar cualquier combinación en progreso
                        self.current_chord.clear()
                        self.chord_in_progress = False
                        continue
                    
                    # Iniciar o continuar formación de combinación
                    if not self.chord_in_progress:
                        # Primera tecla de una nueva combinación
                        self.chord_in_progress = True
                        self.first_press_time = current_time
                        self.current_chord.clear()
                        if self.debug_mode:
                            print("🎵 Iniciando nueva combinación...")
                    
                    # Agregar nuevas teclas a la combinación actual
                    self.current_chord.update(newly_pressed)
                    self.last_press_time = current_time
                    
                    # Mostrar progreso de la combinación
                    if self.debug_mode:
                        chord_letters = [PIN_MAP.get(pin, "?") for pin in self.current_chord 
                                       if pin in PIN_MAP and pin != board.GP10]
                        print(f"🔤 Combinación actual: {', '.join(sorted(chord_letters))}")
                
                # ==================== FASE 2: DETECCIÓN DE TECLAS LIBERADAS ====================
                if newly_released:
                    self.last_press_time = current_time  # Actualizar tiempo de última actividad
                    
                    if self.debug_mode and self.chord_in_progress:
                        remaining_pressed = len(currently_pressed)
                        print(f"📉 Teclas restantes presionadas: {remaining_pressed}")
            
            # ==================== FASE 3: PROCESAMIENTO DE COMBINACIÓN COMPLETA ====================
            if self.chord_in_progress:
                time_since_last_activity = current_time - self.last_press_time
                time_since_start = current_time - self.first_press_time
                
                # Condiciones para procesar la combinación:
                # 1. No hay teclas presionadas Y han pasado 300ms desde la última actividad
                # 2. O han pasado 800ms desde que comenzó la combinación (timeout de seguridad)
                should_process = (
                    (len(currently_pressed) == 0 and time_since_last_activity >= self.chord_stability_time) or
                    (time_since_start >= self.chord_formation_time)
                )
                
                if should_process and self.current_chord:
                    if self.debug_mode:
                        print("✨ Procesando combinación completa...")
                    
                    self.process_chord(self.current_chord.copy())
                    
                    # Resetear estado para la siguiente combinación
                    self.current_chord.clear()
                    self.chord_in_progress = False
                    self.shift_pressed = False
                    
                    if self.debug_mode:
                        print("🔄 Listo para nueva combinación")
                        print("-" * 40)
            
            # Pequeña pausa para no sobrecargar el procesador
            time.sleep(0.01)

# Diccionario estenográfico español CORREGIDO - Solo letras disponibles y sin acentos
# Letras disponibles: A, B, C, D, E, F, G, I, J, L, M, N, O, P, Q, R, S, T, U
# Letras NO disponibles: H, K, V, W, X, Y, Z, Ñ
# Formato: 'combinación_de_letras_ordenadas': 'palabra_resultante'
STENOGRAPH_DICT = {
    # Artículos y palabras muy comunes
    'EL': 'el',
    'AL': 'la',
    'DE': 'de',
    'EQ': 'que',
    'A': 'a',
    'EN': 'en',
    'NU': 'un',
    'ANU': 'una',
    'ES': 'es',
    'SE': 'se',
    'NO': 'no',
    'ET': 'te',
    'LO': 'lo',
    'LE': 'le',
    'AD': 'da',
    'SU': 'su',
    'OPR': 'por',
    'NOS': 'son',
    'CNO': 'con',
    'AAPR': 'para',
    'OS': 'los',
    'AL': 'las',
    'DEL': 'del',
    
    # Pronombres
    'EM': 'me',
    'SU': 'su',
    'SUS': 'sus',
    'EOS': 'eos',
    'ASU': 'sus',
    'IN': 'mi',
    'IT': 'ti',
    'EELL': 'elle',
    
    # Verbos comunes (solo sin H, V, Y, Z, Ñ)
    'ERS': 'ser',
    'AEST': 'esta',
    'ESTA': 'estar',
    'RI': 'ir',
    'CEDIR': 'decir',
    'CEID': 'dice',
    'ACDOR': 'dar',
    'EENRT': 'tener',
    'EENT': 'tiene',
    'AEGTU': 'gustar',
    'ABEELR': 'saber',
    'DEOP': 'poder',
    'EEPRU': 'queree',
    'ACEPR': 'parecer',
    'AEJRR': 'llegar',
    'AIPR': 'pasar',
    'AEGU': 'quedar',
    'EER': 'creer',
    'ADEJR': 'dejar',
    'CENO': 'conocer',
    'ABEJRRT': 'trabajar',
    'EGIRU': 'seguir',
    'ACENRT': 'encontrar',
    'EMNOR': 'poner',
    'ACEIPNRS': 'aparecer',
    'AEMNORT': 'mostrar',
    'AEMRTR': 'tratar',
    'AACBRR': 'acabar',
    'AACLNR': 'alcanzar',
    'AEGNR': 'ganar',
    'ADENR': 'perder',
    'AABCR': 'cambiar',
    
    # Sustantivos comunes (solo letras disponibles)
    'ACAS': 'casa',
    'EIMPT': 'tiempo',
    'ADI': 'dia',
    'OMBRM': 'nombre',
    'AEGLRU': 'lugar',
    'AEFMOR': 'forma',
    'ADM': 'dma',
    'AEPT': 'parte',
    'ADNOR': 'orden',
    'AEMNOT': 'momento',
    'ACOS': 'cosa',
    'ACEOS': 'caso',
    'EEGNRT': 'gente',
    'AEMNOS': 'semana',
    'EMS': 'mes',
    'AGINOP': 'opinion',
    'AEGRU': 'grupo',
    'AEMNOR': 'manera',
    'EFIN': 'fin',
    'ADIR': 'ida',
    'AELNTU': 'cuenta',
    'AEORS': 'razon',
    'AGURU': 'grupo',
    'ADEIR': 'padre',
    'ADEM': 'madre',
    'AEMNOS': 'personas',
    'ACEIM': 'amigo',
    'ACEIJMR': 'mujer',
    'AGUUA': 'agua',
    'ACERN': 'carne',
    'ADEM': 'mader',
    'AEIJM': 'mejor',
    'AEGR': 'gran',
    'EGNR': 'grande',
    
    # Adjetivos (sin letras problemáticas)
    'BENORU': 'bueno',
    'AENRU': 'buena',
    'AEGLNR': 'largo',
    'AEGLNR': 'gran',
    'EOPR': 'poco',
    'CHMO': 'mucho',
    'EIMPR': 'primer',
    'GLIMO': 'ultimo',
    'EIMSM': 'mismo',
    'ORTO': 'otro',
    'EUV': 'nuevo',
    'EJO': 'cierto',
    'EGORN': 'negro',
    'ABCLON': 'blanco',
    'OJR': 'rojo',
    'EDEVRR': 'verde',
    'AILLMRO': 'amarillo',
    'AILRT': 'alto',
    'ABJOR': 'bajo',
    'AGDELR': 'grande',
    'ENOPRQ': 'pequeno',
    'AEILPT': 'simple',
    'CEFILR': 'facil',
    'DEFICILFR': 'dificil',
    'ELOPNR': 'lleno',
    'CEIOR': 'rico',
    'EOBRP': 'pobre',
    'CEILDTU': 'dulce',
    'AGMOR': 'amargo',
    'ACEILOR': 'calor',
    'EFIOR': 'frio',
    
    # Números básicos (solo disponibles)
    'DOS': 'dos',
    'ERTS': 'tres',
    'ACRTU': 'cuatro',
    'CCINO': 'cinco',
    'EISS': 'seis',
    'EEST': 'siete',
    'CHOO': 'ocho',
    'EUV': 'nueve',
    'DEZ': 'diez',
    
    # Adverbios (sin problemas)
    'AMS': 'mas',
    'EEMNOS': 'menos',
    'BINE': 'bien',
    'ALM': 'mal',
    'AQI': 'aqui',
    'AIA': 'ahi',
    'LIL': 'alli',
    'DNE': 'donde',
    'DNOA': 'cuando',
    'MCO': 'como',
    'IS': 'si',
    'OSL': 'solo',
    'AMP': 'tambien',
    'AUNC': 'nunca',
    'EIMRPS': 'siempre',
    'MU': 'muy',
    'CPOO': 'poco',
    'CHMO': 'mucho',
    'YA': 'antes',
    'ESPUD': 'despues',
    'AGLO': 'luego',
    'PNORTO': 'pronto',
    'AETRM': 'temprano',
    'ADERT': 'tarde',
    'AORA': 'ahora',
    
    # Preposiciones (cuidadosamente seleccionadas)
    'ERTS': 'entre',
    'AAJB': 'bajo',
    'EBORS': 'sobre',
    'IS': 'sin',
    'EDS': 'desde',
    'ARTS': 'tras',
    'ETNA': 'ante',
    'ATNOR': 'contra',
    'GSU': 'segun',
    'AETNR': 'durante',
    'AEMNDIT': 'mediante',
    
    # Conjunciones
    'EP': 'pero',
    'AU': 'aunque',
    'CEPRTO': 'porque',
    'OPR': 'por',
    'ESI': 'sino',
    'AEMS': 'mas',
    
    # Expresiones temporales (sin acentos ni letras problemáticas)
    'OT': 'hoy', # Cambio: escribiremos "hoy" como "ot"
    'ER': 'ayer', # Cambio: escribiremos "ayer" como "er"  
    'AANA': 'manana', # sin ñ
    'ECNO': 'noche',
    'AERT': 'tarde',
    'AANA': 'manana',
    'AORA': 'ahora',
    'AENTS': 'antes',
    'ESPUD': 'despues',
    'AGLO': 'luego',
    'PNORTO': 'pronto',
    'ADETRM': 'temprano',
    
    # Familia (solo sin problemas)
    'ADREP': 'padre',
    'ADEM': 'madre',
    'OIT': 'tio',
    'AIT': 'tia',
    'IMROP': 'primo',
    'AIMP': 'prima',
    'AENOM': 'hermano', # Problema: tiene H - eliminado
    # 'AEHMNR': 'hermana', # Problema: tiene H - eliminado
    
    # Emociones (cuidadosamente seleccionadas)
    'EFLIZ': 'feliz', # Problema: tiene Z - eliminado y reemplazado
    'EEILZF': 'alegre', # Cambio a "alegre"
    'ETSIR': 'triste',
    'AELGR': 'alegre',
    'ADFNO': 'enfadado',
    'EOTCNN': 'contento',
    'CDPR': 'preocupado',
    'ACMO': 'calmado',
    'EORSOVN': 'nervioso', # Problema: tiene V - eliminado
    
    # Acciones cotidianas (solo letras disponibles)
    'EMRO': 'comer',
    'EBBR': 'beber',
    'DOMRIR': 'dormir',
    'ACIMNR': 'caminar',
    'CORRR': 'correr',
    'AHLBR': 'hablar', # Problema: tiene H - eliminado
    'ECHURS': 'escuchar', # Problema: tiene H - eliminado
    'EIRM': 'mirar',
    'ELR': 'leer',
    'BIIRS': 'escribir', # Problema: tiene V en "escribir" - eliminado
    'AAJRR': 'trabajar', # Problema: tiene H en "trabajar" - corregido
    'AGJRU': 'jugar',
    'CNATR': 'cantar',
    'AILBR': 'bailar',
    'APRR': 'comprar',
    'EDENRV': 'vender', # Problema: tiene V - eliminado
    'ARUYD': 'ayudar', # Problema: tiene Y - eliminado
    'APRR': 'aprender',
    'AEREÑNS': 'ensenar', # sin ñ
    
    # Lugares (cuidadosamente revisados)
    'ACAS': 'casa',
    'CLEEGIO': 'colegio',
    'AALP': 'plaza', # Problema: tiene Z - eliminado
    'AELCL': 'calle',
    'AECDUI': 'ciudad',
    'AEBLP': 'pueblo',
    'AIS': 'pais',
    'AOPR': 'parque',
    'AOÑMTN': 'montana', # sin ñ
    'ORI': 'rio',
    'ARM': 'mar',
    'ACDEFMR': 'farmacia',
    'ACEIMRS': 'mercado',
    'ABCNO': 'banco',
    'EAIRST': 'restaurante',
    'EFAC': 'cafe', # sin acento
    'EOLT': 'hotel', # Problema: tiene H - eliminado
    'LOHTE': 'hospital', # Problema: tiene H - eliminado
    'IGLESIA': 'iglesia',
    'AEMOSU': 'museo',
    
    # Colores (solo disponibles)
    'OJR': 'rojo',
    'EGNR': 'negro',
    'ABCLON': 'blanco',
    'AOS': 'rosa',
    'IGRS': 'gris',
    
    # Comida (cuidadosamente seleccionadas)
    'NPA': 'pan',
    'EHCL': 'leche', # Problema: tiene H - eliminado
    'ACERN': 'carne',
    'ESCP': 'pescado',
    'OPLL': 'pollo',
    'ARRZ': 'arroz', # Problema: tiene Z - eliminado
    'AASPT': 'pasta',
    'APS': 'sopa',
    'ADALSA': 'ensalada',
    'AUFRT': 'fruta',
    'AEGLN': 'legumbres',
    'NPA': 'pan',
    'ECH': 'queso', # Problema: tiene H - eliminado
    'AFEC': 'cafe', # sin acento
    'TE': 'te',
    'AGU': 'agua',
    'MO': 'zumo', # Problema: tiene Z - eliminado
    
    # Ropa (solo letras disponibles)
    'AISM': 'camisa',
    'AALNSOT': 'pantalon',
    'AAD': 'falda',
    'EISTDV': 'vestido', # Problema: tiene V - eliminado
    'ABRIG': 'abrigo',
    'ACQETU': 'chaqueta', # Problema: tiene H - eliminado
    'EYJR': 'jersey', # Problema: tiene Y - eliminado
    'SACTE': 'camiseta',
    'PAOST': 'zapatos', # Problema: tiene Z - eliminado
    'ACALZSD': 'calcetines', # Problema: tiene Z - eliminado
    'AGFN': 'gafas',
    'OJL': 'reloj', # Problema: tiene J repetida y estructura rara
    
    # Palabras adicionales muy comunes y útiles
    'TOD': 'todo',
    'ALG': 'algo',
    'NAD': 'nada', # Problema: tiene H en "nada" - no, "nada" está bien
    'LACU': 'cual',
    'ADC': 'cada',
    'ADOL': 'lado',
    'MND': 'mundo',
    'OSTR': 'otros',
    'MIS': 'mis',
    'TUS': 'tus',
    'ESTOS': 'estos',
    'ESOS': 'esos',
    'ESTA': 'esta',
    'ESA': 'esa',
    'MINT': 'mientras',
    'MISM': 'misma',
    'MISMS': 'mismas',
    'MISMS': 'mismos',
    'CAS': 'casi',
    'ALT': 'tal',
    'ALGN': 'algun',
    'ALGN': 'alguna',
    'ALGNS': 'algunos',
    'ALGNS': 'algunas',
    'NING': 'ningun',
    'NING': 'ninguna'
}

# Crear e iniciar el teclado estenógrafo
if __name__ == "__main__":
    try:
        stenograph = StenographKeyboard()
        stenograph.run()
    except Exception as e:
        print(f"Error: {e}")
        print("Reiniciando en 5 segundos...")
        time.sleep(5)
