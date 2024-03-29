# Prueba Beitech
Este proyecto contiene los entregables de la prueba indicada por Beitech

## Documentación de los método del API REST
### Clase PruebaOrderRestController
Esta clase, contiene los métodos API REST requeridos por la prueba de Beitech.
Esta clase se puede encontrar en la siguiente ruta:
```bash
--src
    |--main
           |--java
	         |--com
	              |--example
		               |--prueba
			               |--controller
					           |--PruebaOrderRestController.java
```
#### Propiedades de la clase PruebaOrderRestController (repositorios)
Las propiedades de la clase, contienen los repositorios de cada uno modelos (clases) de la aplicación, y permiten generar la conexión con la base de datos.
```bash
@RestController
public class PruebaOrderRestController {
	@Autowired
	private OrderRepository orderRepo;
	@Autowired
	private OrderDetailRepository orderDetailRepo;
	@Autowired
	private CustomerRepository customerRepo;
	@Autowired
	private ProductRepository productRepo;
	@Autowired
	private CustomerProductRepository customerProductRepo;
```
Los repositorios son una extensión de la interfaz `JpaRepository`, y se conectan a la tabla de la base de datos, dependiendo de la clase (entidad) que se incluya en su definición. 

El siguiente es un ejemplo de Repositorio para la clase `Order`:
```bash
public interface OrderRepository extends JpaRepository<Order, Integer> {

}
```
Los repositorios se encuentran en la siguiente ruta:

```bash
--src
    |--main
           |--java
	         |--com
	              |--example
		               |--prueba
			               |--repo
					     |--OrderRepository.java
					     :
					     |-- (Otros repositorios)
```

##### Entidades de un repositorio
Cada repositorio corresponde a una entidad, donde cada entidad se define como una clase.

El siguiente es un ejemplo de la entidad `orden`.

```bash
	@Data // Crea todos los métodos getters, setters, equals, hash y toString, en función de los campos.
	@Entity(name = "Order") // Prepara este objeto para almacenarlo en un almacén de datos basado en JPA
	@Table(name = "\"order\"") // Se indica el nombre de la tabla de la base de datos a la que corresponde. Se utiliza \"order\" que representa el nombre "order" (entre comillas), ya que palabra order es una palabra reservada de mysql.
	public class Order{
		@Id // Indica que es la clave principal al proveedor JPA
		@GeneratedValue(strategy = GenerationType.IDENTITY) // Indica que se llena automáticamente por el proveedro JPA
		@Column(name = "\"order_id\"") // Indica la información de la columna en la base de datos
		private int orderId;
		@ManyToOne
		@JoinColumn(name="\"customer_id\"")
		private Customer customer;
		@Column(name = "\"creation_date\"")
		@Temporal(TemporalType.DATE)
		private Date creationDate;
		@Column(name = "\"delivery_address\"", length = 191)
		private String deliveryAddress;
		@Column(name = "\"total\"", precision = 15, scale = 2)
		private double total;

		public Order(){}

		public Order(Customer customer, Date creationDate, String deliveryAddress, double total){
			this.customer = customer;
			this.creationDate = creationDate;
			this.deliveryAddress = deliveryAddress;
			this.total = total;
		}
	}
```
Las entidades se encuentran en la siguiente ruta:

```bash
--src
    |--main
           |--java
	         |--com
	              |--example
		               |--prueba
			               |--model
					      |--Order.java
					      :
					      |-- (Otras entidades)
```

#### Método allBetweenPerCustomer() de la clase PruebaOrderRestController
El método `allBetweenPerCustomer()` lista las órdenes de un cliente por un rango de fechas.

El detalle del funcionamiento se puede encontrar en la documentación del método.
```bash
	@RequestMapping(value="/pruebaorder" , method=RequestMethod.GET)
	@ResponseBody 
	/**
	* Se crea una lista con todas las órdenes a partir del cliente, y un intervalo de fechas.
	* Si no se envían parámetros el listado no filtra las órdenes (envía el listado completo).
	* Los dos parámetros deben tener el siguiente formato de fecha "yyyy-MM-dd".
	* @param customerId El parámetro customerId indica el ID del cliente que generó los reportes.
	* @param fromDate El parámetro fromDate indica la fecha desde la cuál se generará el reporte de órdenes.
	* @param toDate El parámetro toDate indica la fecha hasta la cuál se generará el reporte de órdenes.
	* @return El método retorna la lista de detalles de órden 
	*/
	List<OrderDetail> allBetweenPerCustomer(
		// Se indica el nómbre del parámetro
		@RequestParam
		(value = "id", required = false)
		Integer customerId,
		// Se indica el nómbre del parámetro
		@RequestParam
		(value = "from", required = false) 
		// Se indica el formato de la fecha
		@DateTimeFormat
		(pattern="yyyy-MM-dd") Date fromDate, 
		@RequestParam
		(value = "to", required = false) 
		@DateTimeFormat
		(pattern="yyyy-MM-dd") Date toDate) {
		//Se obtienen todas la fechas
		return orderDetailRepo.findAll()
				.stream() // se crea un stream
				// se filtran por fecha
				.filter(orderDtl -> {
					// Se obtiene el ID del cliente de al órden.
					int cstmrId = orderDtl.getOrder().getCustomer().getCustomerId();
					// Se obtiene la fecha de creación de la órden
					Date crtnDt = orderDtl.getOrder().getCreationDate();
					// Se indica que si el filtro no existe, no habría por cliente
					// Se aplica el filtro si el valor sí existe 
					return ( customerId == null ? true : customerId == cstmrId )
							// Se indica que si el valor inferior no existe, no habría filtro inferior
							// Se aplica el filtro si el valor sí existe 
							&& (fromDate == null ? true : crtnDt.compareTo(fromDate) >= 0)
							// Se indica que si el valor superio no existe, no habría filtro inferior
							// Se aplica el filtro si el valor sí existe
							&& (toDate == null ? true : crtnDt.compareTo(toDate) <= 0); 
				} )
				.collect(Collectors.toList()); // Se crea una lista
	}
```

#### Método newOrder() de la clase PruebaOrderRestController
El método `newOrder()` crea una órden para un cliente con hasta máximo 5 productos, así sean el mismo producto. Se tiene en cuenta sólo algunos productos están permitidos por cada cliente. 

El detalle del funcionamiento se puede encontrar en la documentación del método.

```bash
	@PostMapping("/pruebaorder")
	/**
	 * Se crea una nueva órden del objeto OrderRequestModel.
	 * Se envía el customerID tipo "int"
	 * Se envía la deliveryAddress tipo "String"
	 * Envía una lista de objetos "products" con
	 * - productId tipo "int"
	 * - quantity tipo "int"
	 *
	 * DE LA FORMA
	 *
	 * {"customerId": cliente (int), "deliveryAddress": direccion (String), "products": [
	 *	{productId: producto1 (int), "quantity": cantidad1 (int)},
	 *	{productId: producto2 (int), "quantity": cantidad2 (int)},
	 *	...
	 *	{productId: productoN (int), "quantity": cantidad3 ()}
	 * ]
	 * }
	 * 
	 * @param newOrderReq El parámetro newOrderReq contiene los valores de la nueva órden.
	 * @return Se retorna la lista con el detalle de la nueva órden, por cada producto.
	 */
	List<OrderDetail> newOrder(@RequestBody OrderRequestModel newOrderReq) {
		// Se crea la "order" para el cliente indicado
		// Se crea "customer" para la orden
		Customer customer = customerRepo
				// Se obtiene "customer" a partir de "customerId"				
				.findById(newOrderReq.getCustomerId())
				// Si el cliente no existe se crea una excepción para indicar que el cliente no existe 
				.orElseThrow(() -> new CustomerNotFoundException(newOrderReq.getCustomerId()));
		
		// Se crea la fecha "creationDate" del pedido a partir de la fecha actual
		Date creationDate = Calendar.getInstance().getTime();
		// Se crea la la dirección "deliveryAddress" a partir de la obtenida en el Request
		String deliveryAddress = newOrderReq.getDeliveryAddress();
		
		// Se crea la órden "order" a partir de los datos obtenidos.
		// El valor "total" se crea con el valor de 0 (posteriormente se reemplazará)
		Order order = new Order(customer, creationDate, deliveryAddress, 0);
		
		// Se optiene la lista de productos con Id del producto y cantidades
		List<ProductQuantityRequestModel> productQuantities = newOrderReq.getProducts();
		
		// Se crea una lista "OrderDetail" por cada producto de "products" dentro de "newOrderReq" obtenida en el Request
		// La lista no tiene en cuenta los "productId" que no se encuentran en la base de datos.
		// La lista no tiene en cuenta los productos que nos están asociados al cliente "customer".
		List<OrderDetail> ordersDetail = productQuantities.stream() // Se convierte en tipo Stream
				// Se crea un filtro para descartar los "productId" que no existen en la base de datos.
				.filter(pQ -> productRepo
						.findById(pQ.getProductId())
						.isPresent())
				// Se crea un filtro para descartar los "product" que no están relacionados con el "customer".
				// Para esto se revisa la tabla "customer_product"
				.filter(pQ -> customerProductRepo
						.findById(new CustomerProductId(customer,productRepo.getOne(pQ.getProductId())))
						.isPresent())
				// Se obtiene una lista de "OrderDetail"
				.map(pQ -> {
					// Se obtiene cada producto a partir de cada "productId" válido de la lista del Request
					Product product = productRepo.getOne(pQ.getProductId());
					// Se obtiene la cantidad de cada próducto válido de la lista Request
					int quantity = pQ.getQuantity();
					// Se crea el OrderDetail a partir de la orden, el producto y la cantidad
					// Los otros valores de OrdenDetail, como "productDescription" se obtienen directamente de "product"
					// Esto se puede verificar en el constructor de "OrderDetail"
					return new OrderDetail(order, product, quantity);
			})
		.collect(Collectors.toList()); // Se convierte a tipo List
		
		// Se obtiene la cantidad total de productos para verificar que no supere el valor de 5
		int totalQnt = ordersDetail
				.stream() // Se conviete a tipo Stream
				// Se obtiene la cantidad por producto.
				.map(ordrsDtl -> ordrsDtl.getQuantity())
				// Se realiza la suma de todos los elementos
				.reduce( 0, (sbtl, elmnt) -> sbtl + elmnt);

		// Se verfifica si la cantidad de productos en la orden se encuentra entre 0 y 5
		if(0 <= totalQnt && totalQnt <= 5) {
			// Se obtiene el costo total de la órden
			double total = ordersDetail
					.stream()
					// Se obtiene el costo de cada producto (quantity * price)
					.map(ordrsDtl -> ordrsDtl.getQuantity() * ordrsDtl.getPrice())
					// Se obtiene el costo total de los productos.
					.reduce((double) 0, (sbtl, elmnt) -> sbtl + elmnt);
			
			// Se incluye el costo total dentro de la órden.
			order.setTotal(total);
			
			// Se almacena la órden "order" dentro de la base de datos.
			orderRepo.save(order);
			
			// Se ajusta la órden por cada producto "orderDetail" incluyendo "order" con el valor "total" actualizado.
			return ordersDetail.stream() // Se convierte a Stream
					.map(ordrsDtl -> {
						// Se incluye la órden "order" con "total" actualizado
						ordrsDtl.setOrder(order);
						// Se guarda "orderDetail" dentro de la base de datos
						return orderDetailRepo.save(ordrsDtl);
						})
					.collect(Collectors.toList()); // Se conviete a List
			
		}else { // Si no está dentro del rango de 0 a 5 se envía una excepción.
			throw new ProductQuantityOrderRangeException(totalQnt);
		}
	    
	}
	
}
```
## Diagrama de Clases

Figura 1. Diagrama de Clases
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/classDiagram.png)

## Servicios web REST (BackEnd)
Para ingresar a los servicios web REST se debe ingresar la dirección URL `http://<hostname>:8080/pruebaorder`.

### Crear una órden (POST)

Para crear una órden se debe enviar un POST Request cuyo cuerpo tenga la forma del objeto JSON:

```bash
{ 
  "customerId": id_del_cliente_1 , 
  "deliveryAddress": "... alguna direccion ...", 
  "products": [ 
		{ 
		"productId" : id_del_producto_1, 
		"quantity": cantidad_del_producto_1
		}, 
		{ 
		"productId" : 13, 
		"quantity": 1
		}
	      ]
}
```

**Ejemplo 1**

Figura 2. POST Request
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/post1.png)

Figura 3. Response
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/post2.png)

### Listar órdenes (GET)

Para crear una órden se debe enviar un GET Request cuyos parámetros sean los siguientes:

```bash
localhost:8080/pruebaorder?from=<fecha desde>&to=<fecha hasta>&id=<id del cliente>
```
**Ejemplo 2**

Figura 4. GET Request
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/get1.png)

Figura 5. Response
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/get2.png)

## Aplicación AngularJS (FrontEnd)

Para ingresar a la aplicación HTML se debe ingresar la dirección URL `http://<hostname>:8080/`.

Figura 6. Aplicación en AngularJS
![imagen](https://raw.githubusercontent.com/luisca1985/prueba-beitech/master/assets/app.png)
