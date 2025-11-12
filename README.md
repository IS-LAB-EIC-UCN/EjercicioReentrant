# 🧩 Ejercicios de Concurrencia en Java

## Ejercicio 1: Contador Seguro con ReentrantLock

### 🎯 Objetivo
Implementar un contador compartido entre varios hilos, asegurando la exclusión mutua mediante `ReentrantLock`.

### 🧭 Contexto
Cuando varios hilos acceden y modifican una misma variable, pueden producirse **condiciones de carrera**. Para evitarlo, se usa un mecanismo de bloqueo.  
`ReentrantLock` es una clase que permite controlar manualmente cuándo un hilo entra o sale de una sección crítica, con la ventaja de ser **reentrante**: el mismo hilo puede adquirir el bloqueo más de una vez sin bloquearse.

### 🧩 Código de ejemplo

```java
import java.util.concurrent.locks.ReentrantLock;

class ContadorSeguro {
    private int valor = 0;
    private final ReentrantLock bloqueo = new ReentrantLock(true); // justo (fair)

    public void incrementar() {
        bloqueo.lock();
        try {
            valor++;
            System.out.println(Thread.currentThread().getName() + " incrementó a " + valor);
        } finally {
            bloqueo.unlock();
        }
    }

    // Variante opcional: intento no bloqueante
    public boolean intentarIncrementar() {
        if (bloqueo.tryLock()) {
            try {
                valor++;
                System.out.println(Thread.currentThread().getName() + " incrementó (tryLock) a " + valor);
                return true;
            } finally {
                bloqueo.unlock();
            }
        } else {
            return false; // no se pudo adquirir el bloqueo
        }
    }

    public int obtenerValor() {
        return valor;
    }
}

public class Aplicacion {
    public static void main(String[] args) throws InterruptedException {
        ContadorSeguro contador = new ContadorSeguro();

        Thread hilo1 = new Thread(() -> {
            for (int i = 0; i < 5; i++) contador.incrementar();
        }, "Hilo-1");

        Thread hilo2 = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                if (!contador.intentarIncrementar()) {
                    contador.incrementar();
                }
            }
        }, "Hilo-2");

        hilo1.start();
        hilo2.start();

        hilo1.join();
        hilo2.join();

        System.out.println("Valor final del contador: " + contador.obtenerValor());
    }
}
```

### ✅ Puntos clave
- Usar `lock()` y `unlock()` siempre dentro de un bloque `try/finally`.
- `ReentrantLock` permite:
    - Intentos no bloqueantes (`tryLock()`).
    - Bloqueo justo (`new ReentrantLock(true)`).
    - Bloqueo interrumpible (`lockInterruptibly()`).
- Evitar condiciones de carrera en secciones críticas.


---

## Ejercicio 2: Buffer Acotado con await() y signal()

### 🧭 Contexto
En el clásico problema **Productor–Consumidor**, varios hilos producen y consumen datos de un **buffer compartido** de capacidad limitada.  
Cuando el buffer está **lleno**, los productores deben esperar; cuando está **vacío**, los consumidores deben esperar.

En Java, esto puede implementarse usando `ReentrantLock` y `Condition`:
- `await()` suspende un hilo hasta que se cumpla una condición.
- `signal()` o `signalAll()` despiertan hilos que estaban esperando.

### 🎓 Objetivo didáctico
- Aplicar sincronización mediante `await()` y `signal()`.
- Controlar correctamente los estados *lleno* y *vacío* del buffer.
- Evitar condiciones de carrera o interbloqueo.

---

### 🧩 Código esqueleto (para completar)

```java
import java.util.ArrayDeque;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

class BufferAcotado<T> {
    private final Queue<T> cola;
    private final int capacidad;

    private final ReentrantLock bloqueo = new ReentrantLock(true);
    private final Condition noLleno = bloqueo.newCondition();
    private final Condition noVacio = bloqueo.newCondition();

    public BufferAcotado(int capacidad) {
        if (capacidad <= 0) throw new IllegalArgumentException("capacidad debe ser > 0");
        this.capacidad = capacidad;
        this.cola = new ArrayDeque<>(capacidad);
    }

    public void poner(T elemento) throws InterruptedException {
        bloqueo.lock();
        try {
            // TODO: Esperar mientras la cola esté llena
            // while (cola.size() == capacidad) { noLleno.await(); }

            cola.add(elemento);

            // TODO: Notificar que ahora el buffer no está vacío
            // noVacio.signal();
        } finally {
            bloqueo.unlock();
        }
    }

    public T tomar() throws InterruptedException {
        bloqueo.lock();
        try {
            // TODO: Esperar mientras la cola esté vacía
            // while (cola.isEmpty()) { noVacio.await(); }

            T valor = cola.remove();

            // TODO: Notificar que ahora el buffer no está lleno
            // noLleno.signal();

            return valor;
        } finally {
            bloqueo.unlock();
        }
    }

    public int tamañoActual() {
        bloqueo.lock();
        try {
            return cola.size();
        } finally {
            bloqueo.unlock();
        }
    }
}

public class PruebaBuffer {
    public static void main(String[] args) throws InterruptedException {
        BufferAcotado<Integer> buffer = new BufferAcotado<>(2);

        Runnable productor = () -> {
            for (int i = 1; i <= 5; i++) {
                try {
                    buffer.poner(i);
                    System.out.println(Thread.currentThread().getName()
                            + " puso: " + i + " (tamaño=" + buffer.tamañoActual() + ")");
                    Thread.yield();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };

        Runnable consumidor = () -> {
            for (int i = 1; i <= 5; i++) {
                try {
                    Integer v = buffer.tomar();
                    System.out.println(Thread.currentThread().getName()
                            + " tomó: " + v + " (tamaño=" + buffer.tamañoActual() + ")");
                    Thread.yield();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        };

        Thread p1 = new Thread(productor, "Productor-1");
        Thread c1 = new Thread(consumidor, "Consumidor-1");
        Thread c2 = new Thread(consumidor, "Consumidor-2");

        p1.start();
        c1.start();
        c2.start();

        p1.join();
        c1.join();
        c2.join();

        System.out.println("Fin. Tamaño final del buffer = " + buffer.tamañoActual());
    }
}
```