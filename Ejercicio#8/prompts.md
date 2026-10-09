# Ejercicio #8 – Actividad de Semana 11

**Integrantes:**
- Ignacio Hernández 
- Julián Moya

## Descripción

Poner en práctica los tres patrones de diseño estructurales de la semana 11 (Adapter, Decorator y Facade) mediante una actividad en la que había que modificar un programa existente (**TecShop checkout**) con ayuda de un asistente de IA en VS Code. Primero detectamos los tres problemas de diseño y luego escribimos los prompts siguiendo la plantilla (CONTEXTO, TAREA, ESTRUCTURA, REGLAS, TERMINADO) y las restricciones de la actividad, por ejemplo no usar en los prompts los nombres de los patrones. Después de cada cambio verificamos que `Main` siguiera imprimiendo exactamente la salida base.

---

## Mapa de problemas

| # | Dónde (clase.método) | Síntoma: ¿qué va a doler cuando el código cambie? | ¿Quién sabe demasiado? | ¿Qué clase(s) existente(s) dejarían intactas? |
|---|---|---|---|---|
| S1 | `ShippingService.quote()` (caso `"correos"`) | Antes de llamar a `CorreosApi`, el servicio convierte kilos a gramos, la ciudad a código postal y el texto `"₡3750"` a `double`. Si Correos cambia su SDK o agregamos otro proveedor externo, hay que volver a meter conversiones dentro de `ShippingService`. | `ShippingService`, porque conoce los detalles internos de cómo funciona la API de Correos. | `CorreosApi` (no se puede modificar), `ShippingProvider`, `LocalCourier` y los servicios de la tienda. |
| S2 | `CartItem.getPrice()`<br>`CartItem.describe()` | Cada vez que agreguemos un nuevo extra tendremos que modificar `CartItem` y agregar más condiciones `if`, haciendo el código más difícil de mantener. | `CartItem`, porque conoce todos los extras, sus precios y cómo modificar las descripciones. | `Inventory`, `TaxCalculator`, `PaymentGateway`, `ShippingService` y las demás clases de servicios. |
| S3 | `WebCheckout.buy()`<br>`KioskCheckout.buy()` | Ambos métodos repiten los mismos pasos de compra. Si cambia el proceso, tendremos que modificar las dos clases y podríamos generar inconsistencias. | `WebCheckout` y `KioskCheckout`, porque conocen y coordinan todos los servicios necesarios para realizar una compra. | `Inventory`, `TaxCalculator`, `PaymentGateway`, `InvoicePrinter`, `ShippingService` y los demás servicios. |

---

## S1: Patrón de diseño Adapter

`CorreosApi` es una librería externa que no se puede modificar, y su método `calcularTarifa(int gramos, String codigoPostal)` no coincide con la interfaz que el programa ya tenía para los envíos, `ShippingProvider.quote(double kg, String city)`. Creamos una clase nueva, `CorreosShipping`, que implementa `ShippingProvider` y guarda un objeto `CorreosApi`. Por dentro, su método `quote` traduce los datos (kilos a gramos, ciudad a código postal y `"₡..."` a `double`) y delega el cálculo a la API externa. Así, `ShippingService` solo trabaja con nuestra interfaz y ya no conoce los detalles de Correos.

### Prompt 1

```text
CONTEXTO: ShippingProvider.java y CorreosApi.java (adjuntos).
TAREA: Creá una clase nueva CorreosShipping en el paquete tecshop.
ESTRUCTURA: CorreosShipping implementa ShippingProvider. Tiene un único
atributo private final CorreosApi api, que recibe como parámetro en su
constructor público. Por ahora, el método quote(double kg, String city)
solo lanza UnsupportedOperationException.
REGLAS: No modifiques CorreosApi, ShippingProvider ni ningún otro archivo.
Sin librerías nuevas. Paquete tecshop.
TERMINADO: Compila. Mostrame solo el archivo nuevo CorreosShipping.java
completo, sin explicaciones extra.
```

**Salida de la IA:**

`CorreosShipping.java`

```java
package tecshop;

public class CorreosShipping implements ShippingProvider {
    private final CorreosApi api;

    public CorreosShipping(CorreosApi api) {
        this.api = api;
    }

    @Override
    public double quote(double kg, String city) {
        throw new UnsupportedOperationException();
    }
}
```

### Prompt 2

```text
CONTEXTO: CorreosShipping.java, CorreosApi.java y ShippingService.java (adjuntos).
TAREA: Implementá el método quote de CorreosShipping. Solo cambia CorreosShipping.
ESTRUCTURA: quote(double kg, String city) debe:
  1. Convertir kg a gramos como int: (int) Math.round(kg * 1000).
  2. Convertir la ciudad a código postal con un método private
     String toPostalCode(String city), que usa un switch con los mismos
     casos que hoy tiene ShippingService: "San Jose"->"10101",
     "Alajuela"->"20101", "Cartago"->"30101", "Limon"->"70101";
     cualquier otra ciudad lanza IllegalArgumentException("Unknown city: " + city).
  3. Delegar en api.calcularTarifa(gramos, codigoPostal).
  4. Convertir el String que retorna (ej. "₡3750") a double, quitando "₡",
     y retornarlo.
REGLAS: No modifiques CorreosApi, ShippingService ni ningún otro archivo.
No cambies la firma de quote. Sin librerías nuevas.
TERMINADO: Compila. Mostrame solo CorreosShipping.java completo.
```

**Salida de la IA:**

`CorreosShipping.java`

```java
package tecshop;

public class CorreosShipping implements ShippingProvider {
    private final CorreosApi api;

    public CorreosShipping(CorreosApi api) {
        this.api = api;
    }

    @Override
    public double quote(double kg, String city) {
        int gramos = (int) Math.round(kg * 1000);
        String codigoPostal = toPostalCode(city);
        String raw = api.calcularTarifa(gramos, codigoPostal);
        return Double.parseDouble(raw.replace("₡", ""));
    }

    private String toPostalCode(String city) {
        switch (city) {
            case "San Jose":
                return "10101";
            case "Alajuela":
                return "20101";
            case "Cartago":
                return "30101";
            case "Limon":
                return "70101";
            default:
                throw new IllegalArgumentException("Unknown city: " + city);
        }
    }
}
```

### Prompt 3

```text
CONTEXTO: ShippingService.java, WebCheckout.java, KioskCheckout.java,
CorreosShipping.java, LocalCourier.java y ShippingProvider.java (adjuntos).
TAREA: Hacer que ShippingService dependa solo de ShippingProvider.
ESTRUCTURA:
  - ShippingService tiene dos atributos private final ShippingProvider:
    local y correos. Los recibe por un constructor público
    ShippingService(ShippingProvider local, ShippingProvider correos).
    Elimina los atributos actuales de tipo LocalCourier y CorreosApi.
  - quote(String provider, double kg, String city) mantiene su firma:
    si provider es "local" delega en local.quote(kg, city); si es "correos"
    delega en correos.quote(kg, city); si no, lanza
    IllegalArgumentException("Unknown provider: " + provider).
    Elimina de ShippingService toda conversión de gramos, códigos postales y "₡".
  - En WebCheckout y KioskCheckout cambiá ÚNICAMENTE la línea que crea
    ShippingService por:
    new ShippingService(new LocalCourier(), new CorreosShipping(new CorreosApi()))
REGLAS: No modifiques CorreosApi, ShippingProvider, LocalCourier,
CorreosShipping ni ningún otro archivo. No cambies nada más en los checkouts.
TERMINADO: Compila y procura que Main imprime exactamente la salida base.
Mostrame solo las líneas cambiadas de cada archivo modificado.
```

**Salida de la IA:**

`ShippingService.java`

```diff
-    private final LocalCourier local = new LocalCourier();
-    private final CorreosApi correos = new CorreosApi();
+    private final ShippingProvider local;
+    private final ShippingProvider correos;
+
+    public ShippingService(ShippingProvider local, ShippingProvider correos) {
+        this.local = local;
+        this.correos = correos;
+    }
-            int gramos = (int) Math.round(kg * 1000);
-            String codigoPostal;
-            switch (city) {
-                case "San Jose": codigoPostal = "10101"; break;
-                case "Alajuela": codigoPostal = "20101"; break;
-                case "Cartago":  codigoPostal = "30101"; break;
-                case "Limon":    codigoPostal = "70101"; break;
-                default: throw new IllegalArgumentException("Unknown city: " + city);
-            }
-            String raw = correos.calcularTarifa(gramos, codigoPostal);
-            return Double.parseDouble(raw.replace("₡", ""));
+            return correos.quote(kg, city);
```

`WebCheckout.java`

```diff
-        ShippingService shipping = new ShippingService();
+        ShippingService shipping = new ShippingService(new LocalCourier(), new CorreosShipping(new CorreosApi()));
```

`KioskCheckout.java`

```diff
-        ShippingService shipping = new ShippingService();
+        ShippingService shipping = new ShippingService(new LocalCourier(), new CorreosShipping(new CorreosApi()));
```

---

## S2: Patrón de diseño Decorator

`CartItem` manejaba los extras (envoltura de regalo, seguro y envío exprés) con banderas booleanas y cadenas de `if`, así que cada extra nuevo obligaba a modificar la clase. Creamos una interfaz `Item` con `getPrice()` y `describe()`, que implementan tanto el producto base (`CartItem`) como una clase abstracta `ItemExtra`. `ItemExtra` guarda otro `Item` y delega en él. Cada extra (`GiftWrap`, `Insurance`, `Express`) extiende `ItemExtra` y le suma su costo y su texto al resultado del objeto que contiene. Así los extras se pueden apilar en tiempo de ejecución desde `Main`, y agregar uno nuevo solo requiere crear una clase. El orden de apilado importa: en el teclado se aplica primero la envoltura y luego el seguro, porque el seguro es un 5 % del precio acumulado.

### Prompt

```text
CONTEXTO: En TecShop, CartItem utiliza booleanos y condiciones para manejar los
extras de los productos. Queremos que cada extra sea independiente y pueda
combinarse con otros en tiempo de ejecución. Archivos: Cart, CartItem, Main,
InvoicePrinter, ShippingProvider
TAREA: Reorganizá la representación de los productos y sus extras para que puedan
combinarse dinámicamente sin modificar las clases existentes al agregar nuevas
características.
ESTRUCTURA:
1. Creá una interfaz Item con los métodos double getPrice() y String describe().
2. Hacé que CartItem implemente Item, conservando únicamente el nombre y precio
   base, sin booleanos.
3. Creá una clase abstracta ItemExtra que implemente Item y almacene un
   protected final Item item, recibido mediante el constructor. Sus métodos deben
   delegar al objeto almacenado.
4. Creá tres clases que extiendan ItemExtra:
   * GiftWrap: suma ₡1.500 y agrega " + gift wrap" a la descripción.
   * Insurance: suma un 5 % del precio acumulado y agrega " + insurance".
   * Express: suma ₡2.500 y agrega " + express".
5. Cada clase debe calcular su resultado a partir de los métodos del objeto que
   contiene.
6. Modificá Cart para almacenar objetos Item en lugar de CartItem y ajustá
   InvoicePrinter para trabajar con ellos.
7. Actualizá Main para combinar los extras mediante constructores, conservando los
   productos, precios y orden originales. En el teclado, aplicá primero envoltura
   y después seguro.
REGLAS: Mantené el paquete tecshop, no agregués librerías externas y no modifiqués
los servicios de inventario, pagos, impuestos ni envíos. No utilicés booleanos para
manejar los extras ni cambiés los cálculos, nombres o formato de las facturas. Cada
extra debe poder contener otro objeto Item. Debe ser posible agregar un cuarto
extra creando únicamente una clase nueva, sin modificar las existentes.
TERMINADO: El proyecto debe compilar y Main debe imprimir exactamente la salida
original, incluyendo precios, descripciones y orden. Mostrame únicamente los
archivos nuevos o modificados con su contenido completo e indicá si verificaste la
compilación y la salida.
```

**Salida de la IA:**

`Item.java`

```java
package tecshop;

public interface Item {
    double getPrice();

    String describe();
}
```

`CartItem.java`

```java
package tecshop;

public class CartItem implements Item {
    private final String name;
    private final double basePrice;

    public CartItem(String name, double basePrice) {
        this.name = name;
        this.basePrice = basePrice;
    }

    @Override
    public double getPrice() {
        return basePrice;
    }

    @Override
    public String describe() {
        return name;
    }
}
```

`ItemExtra.java`

```java
package tecshop;

public abstract class ItemExtra implements Item {
    protected final Item item;

    protected ItemExtra(Item item) {
        this.item = item;
    }

    @Override
    public double getPrice() {
        return item.getPrice();
    }

    @Override
    public String describe() {
        return item.describe();
    }
}
```

`GiftWrap.java`

```java
package tecshop;

public class GiftWrap extends ItemExtra {
    public GiftWrap(Item item) {
        super(item);
    }

    @Override
    public double getPrice() {
        return item.getPrice() + 1500;
    }

    @Override
    public String describe() {
        return item.describe() + " + gift wrap";
    }
}
```

`Insurance.java`

```java
package tecshop;

public class Insurance extends ItemExtra {
    public Insurance(Item item) {
        super(item);
    }

    @Override
    public double getPrice() {
        return item.getPrice() * 1.05;
    }

    @Override
    public String describe() {
        return item.describe() + " + insurance";
    }
}
```

`Express.java`

```java
package tecshop;

public class Express extends ItemExtra {
    public Express(Item item) {
        super(item);
    }

    @Override
    public double getPrice() {
        return item.getPrice() + 2500;
    }

    @Override
    public String describe() {
        return item.describe() + " + express";
    }
}
```

`Cart.java`

```java
package tecshop;

import java.util.ArrayList;
import java.util.List;

public class Cart {
    private final List<Item> items = new ArrayList<>();
    private final double weightKg;

    public Cart(double weightKg) {
        this.weightKg = weightKg;
    }

    public void add(Item item) {
        items.add(item);
    }

    public List<Item> getItems() {
        return items;
    }

    public double getWeightKg() {
        return weightKg;
    }

    public double getSubtotal() {
        double subtotal = 0;
        for (Item item : items) {
            subtotal += item.getPrice();
        }
        return subtotal;
    }
}
```

`InvoicePrinter.java`

```java
package tecshop;

public class InvoicePrinter {
    public void print(String customer, Cart cart, double tax, double shipping, double total) {
        System.out.println("----- INVOICE: " + customer + " -----");
        for (Item item : cart.getItems()) {
            System.out.println("  " + item.describe() + "  " + Money.format(item.getPrice()));
        }
        System.out.println("  Subtotal: " + Money.format(cart.getSubtotal()));
        System.out.println("  Tax:      " + Money.format(tax));
        System.out.println("  Shipping: " + Money.format(shipping));
        System.out.println("  TOTAL:    " + Money.format(total));
    }
}
```

`Main.java`

```java
package tecshop;

public class Main {
    public static void main(String[] args) {
        Cart webCart = new Cart(2.3);
        webCart.add(new Express(new GiftWrap(new CartItem("Laptop stand", 25000))));
        webCart.add(new Insurance(new CartItem("USB-C hub", 18000)));
        webCart.add(new CartItem("Notebook", 3500));
        new WebCheckout().buy(webCart, "Ana", "Cartago");

        System.out.println();

        Cart kioskCart = new Cart(0.8);
        kioskCart.add(new Insurance(new GiftWrap(new CartItem("Mechanical keyboard", 42000))));
        new KioskCheckout().buy(kioskCart, "Luis", "San Jose");
    }
}
```

---

## S3: Patrón de diseño Facade

`WebCheckout.buy()` y `KioskCheckout.buy()` repetían la misma secuencia de pasos sobre varias clases del subsistema de la tienda: reservar inventario, calcular impuestos, cotizar el envío, cobrar e imprimir la factura. Creamos la clase `CheckoutService`, que es dueña de esa secuencia y la ofrece en un solo método, `processPurchase()`. Lo único que cambia entre los dos checkouts (el proveedor de envío y el método de pago) se pasa como parámetro. Cada checkout quedó reducido a una sola llamada, y las clases de servicios de la tienda no se modificaron.

### Prompt

```text
Archivos: WebCheckout, KioskCheckout, Inventory, TaxCalculator, ShippingService,
PaymentGateway, InvoicePrinter, Cart, Money

CONTEXTO: En TecShop, WebCheckout.buy() y KioskCheckout.buy() repiten la misma
secuencia de operaciones para realizar una compra: reservar inventario, calcular
subtotal e impuestos, obtener el costo de envío, procesar el pago e imprimir la
factura. Esto obliga a modificar ambas clases cada vez que cambia el proceso.

TAREA: Centralizá la secuencia de compra en una clase nueva para que los dos tipos
de checkout utilicen un único método, sin duplicar las operaciones.

ESTRUCTURA:
1. Creá una clase CheckoutService dentro del paquete tecshop.
2. Esta clase debe encargarse de coordinar Inventory, TaxCalculator,
   ShippingService, PaymentGateway e InvoicePrinter.
3. Creá un método público processPurchase() que reciba los datos necesarios de la
   compra, incluyendo el carrito, cliente, ciudad y método de pago, respetando los
   tipos y parámetros utilizados actualmente.
4. Dentro de ese método ejecutá las operaciones en el mismo orden original:
   reservar inventario, calcular subtotal e impuestos, obtener el envío, calcular
   el total, realizar el cobro e imprimir la factura.
5. Modificá WebCheckout.buy() y KioskCheckout.buy() para que cada uno realice
   únicamente una llamada a CheckoutService, pasando los datos correspondientes.
6. Los servicios deben crearse y utilizarse dentro de CheckoutService, no dentro de
   las clases de checkout.

REGLAS: Mantené el paquete tecshop, no agregués librerías externas y no modifiqués
Inventory, TaxCalculator, ShippingService, PaymentGateway, InvoicePrinter, Money ni
las clases relacionadas con los productos. Conservá los parámetros, cálculos,
mensajes y orden de ejecución originales. No cambiés la forma en que Main llama a
los métodos buy().

TERMINADO: El proyecto debe compilar y Main debe imprimir exactamente la salida
original. WebCheckout y KioskCheckout no deben crear directamente los servicios de
la tienda y sus métodos buy() deben quedar reducidos a una sola llamada cada uno.
Mostrame únicamente los archivos nuevos o modificados con su contenido completo e
indicá si verificaste la salida.
```

**Salida de la IA:**

`CheckoutService.java`

```java
package tecshop;

public class CheckoutService {
    public void processPurchase(
            Cart cart,
            String customer,
            String city,
            String shippingProvider,
            String paymentMethod) {
        Inventory inventory = new Inventory();
        inventory.reserve(cart);

        TaxCalculator taxCalculator = new TaxCalculator();
        double subtotal = cart.getSubtotal();
        double tax = taxCalculator.tax(subtotal);

        ShippingService shipping = new ShippingService(
                new LocalCourier(),
                new CorreosShipping(new CorreosApi()));
        double shippingCost = shipping.quote(shippingProvider, cart.getWeightKg(), city);

        double total = subtotal + tax + shippingCost;
        PaymentGateway gateway = new PaymentGateway();
        gateway.charge(paymentMethod, total);

        InvoicePrinter printer = new InvoicePrinter();
        printer.print(customer, cart, tax, shippingCost, total);
    }
}
```

`WebCheckout.java`

```java
package tecshop;

// Online store checkout.
public class WebCheckout {
    public void buy(Cart cart, String customer, String city) {
        new CheckoutService().processPurchase(cart, customer, city, "correos", "credit card");
    }
}
```

`KioskCheckout.java`

```java
package tecshop;

// In-store kiosk checkout. Copied from WebCheckout and adjusted.
public class KioskCheckout {
    public void buy(Cart cart, String customer, String city) {
        new CheckoutService().processPurchase(cart, customer, city, "local", "SINPE Movil");
    }
}
```
