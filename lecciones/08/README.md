Lección 08: Campos funcionales, atributos y decoradores avanzados
=================================================================

[TOC]

Campos Related
--------------
En algunos casos es necesario tener disponible el valor de un campo de un Modelo relacionado, como si fuera un campo del Modelo actual, por ejemplo si tenemos los Modelos Persona -> Ciudad -> Pais, podemos adicionar un campo en Persona que nos diga directamente cual es el Pais, basado en la selección de la Ciudad. Para esto definimos un proxy a través de un campo con el atributo `related`.

```python
from openerp import models, fields, api


class pais(models.Model):
    _name = 'mi_modulo.pais'
    _description = 'Pais'

    name = fields.Char('Nombre')


class ciudad(models.Model):
    _name = 'mi_modulo.ciudad'
    _description = 'Ciudad'

    name = fields.Char('Nombre')
    pais_id = fields.Many2one('mi_modulo.pais','País')


class persona(models.Model):
    _name = 'mi_modulo.persona'
    _description = 'Persona'

    ciudad_id = fields.Many2one('mi_modulo.ciudad', 'Ciudad de Residencia')
    pais = fields.Many2one(related='ciudad_id.pais_id')

```

Ud puede modificar el campo `persona->pais_id` y eso cambiará el valor en el registro `ciudad->pais_id`, también automáticamente la interfaz web va a mostrar el país cuando se selecciones la ciudad. Se pueden adicionar otros atributos propios del campo que aplicarian solo al Modelo actual (persona) tales como: string, help, readonly, required y estos no afectarían la definición original del campo.
`
Campos computados
-----------------

Muchas veces los campos tienen valores que deben calcularse de automáticamente, para esto se utiliza el atributo: `compute` y el nombre del método del Modelo que hará el cálculo del valor. Por ejemplo:

```python
from openerp import models, fields, api
import datetime.timedelta

class biblioteca_prestamo(models.Model):
    _name = 'biblioteca.prestamo'
    _description = 'Informacion de prestamo de libros'

    fecha = fields.Datetime(
        'Fecha del Prestamo',
        help="Fecha en la que se presta el libro",
    )
    duracion_dias = fields.Integer(
        'Duración del Prestamo(días)',
        help="Número días por los cuales se presta el libro",
    )
    fecha_devolucion = fields.Datetime(
        'Fecha Devolución',
        compute='_compute_fecha_devolucion',
        help="Fecha de devolución del libro",
    )

    @api.one
    def _compute_fecha_devolucion(self):
        """Calcula la fecha de devolución basado en la fecha inicial y la duración en días del prestamo"""
        fecha = fields.Datetime.from_string(self.fecha)
        self.fecha_devolucion = fecha + timedelta(days=10)

```

Con el código anterior el cálculo se va a realizar cuando se guarde el registro en la base de datos y en la interfaz el campo se va a mostrar como de solo lectura. Si se desea que se pueda también definir manualmente la fecha de devolución, se debe usar el atributo `inverse` y el nombre del método del Modelo que hará el cálculo inverso. Para nuestro ejemplo el método inverson debería calcular la duración en días a partir de la fecha del prestamo y la fecha de devolución.

```python
from openerp import models, fields, api
import datetime.timedelta

class biblioteca_prestamo(models.Model):
    _name = 'biblioteca.prestamo'

    [...]

    fecha_devolucion = fields.Datetime(
        'Fecha Devolución',
        compute='_compute_fecha_devolucion',
        inverse='_compute_inv_fecha_devolucion',
        help="Fecha de devolución del libro",
    )

    [...]

    @api.one
    def _compute_inv_fecha_devolucion(self):
        """Calcula la fecha duración en días del prestamo basado en la fecha de devolución"""
        if self.fecha and self.fecha_devolucion:
            fecha = fields.Datetime.from_string(self.fecha)
            fecha_devolucion = fields.Datetime.from_string(self.fecha_devolucion)
            delta = fecha_devolucion - fecha
            self.duracion_dias = delta.days

```

Por defecto el valor del campo es calculado en el momento en que se carga el registro y no es almacenado en la base de datos, si se desea que se almacene en la base de datos se debe adicionar el atributo `store=True`. :happy_person_raising_one_hand: Cuando se utiliza store=True se recomienda utilizar el decorador depends, para que que se le indique cuando debe ser recalculado el valor del campo basado en los valores de otros campos del modelo, más detalles en la siguiente sección.

**TODO:** Adicionar y explicar como funciona el atributo store=True/False

Decorador: depends
------------------

Cuando se tiene un campo cálculado/computado con el atributo `store=True`, el decorador `@api.depends` permite indicar *cuando* se va a realizar el llamado a la función de cálculo y por ende actualización del valor del campo. Este cuando esta dado por el cambio de otro campo existente en el modelo o en modelos relacionados.
Por ejemplo podemos adicionar un campo computado en *biblioteca.libro* que nos indique si el libro se encuentra prestado o no, revisando que de los prestamos asociados este por lo menos uno en el estado *'prestado'*. El campo se debe actualizar solamente cuando se adicione un prestamo o cuando un prestamo existente cambie de estado.

```python
from openerp import models, fields, api

class biblioteca_libro(models.Model):
    _name = 'biblioteca.libro'

    is_prestado = fields.Boolean(
        'El libro se encuentra prestado?',
        help="Indica si el libro se encuentra prestado actualmente",
        compute='_compute_is_prestado',
        store=False,
        default=False,
    )

    @api.one
    @api.depends('prestamo_ids.state')
    def _compute_is_prestado(self):
        """Revisa si el libro se encuentra prestado o no, revisando si hay por 
        lo menos un prestamo activo"""
        esta_prestado = False
        for prestamo in self.prestamo_ids:
            if prestamo.state == 'prestado':
                esta_prestado = True
                break
        self.is_prestado = esta_prestado


class biblioteca_prestamo(models.Model):
    _name = 'biblioteca.prestamo'

    state = fields.Selection(
        [
            ('prestado', 'Prestado'),
            ('devuelto', 'Devuelto'),
        ],
        'Estado',
        help='Indica el estado actual del prestamo',
        default='prestado',
    )
```

Ejecutando el código anterior en modo debug puede notar que el método `_compute_is_prestado` es ejecutado cuando:

- Se crea un prestamo
- Cuando se modifica un campo del libro que invoque una constraint (ej. 'fecha_publicacion','fecha_compra', 'paginas'), si modifica un campo diferente ej 'name' no se ejecuta el cálculo
- Cuando se cambia el estado de un prestamo existente
- Cuando se modifica el prestamo y se asigna a otro libro; en este último caso ud notará que el método se llama dos veces, uno para actualizar el libro original y el otro para actualizar el nuevo libro asignado.

Se puede también indicar que se ejecute el computo del campo con campos del mismo objeto libro, solo necesita adicionar el nombre del campo que debe invocar la ejecución del cálculo. ej:

```python
    @api.one
    @api.depends('prestamo_ids.state', 'name')
    def _compute_is_prestado(self):
        """Revisa si el libro se encuentra prestado o no, revisando si hay por 
        lo menos un prestamo activo"""
        [....]
```

Decorador: multi
----------------

Decorador: one
--------------

Decorador: model
----------------

Atributo: states
----------------

Manejo de árboles
------------------

    - related
    - depends
    - multi
    - states
    - parent

Related
-------

Los campos related se utiliza cuando el campo es una referencia de un id de otra tabla. Esto es para hacer referencia a la relación de una relación.


**Estructura de definición**

	'nombre_campo_related': fields.related (
		'id_busqueda',
		'campo_busqueda',
		type = "tipo_campo",
		relation = "objeto_negocio_a_relacionar",
		string = "Texto del campo",
		store = False
	)

* **id_busqueda**: nombre de campo id de referencia del objeto de negocio relacionado para realizar la búsqueda.
* **campo_busqueda**: nombre del campo de referencia, el cual devuelve el valor de ese campo según la busqueda realizada por el id_busqueda en el objeto de negocio a relacionar.
* **type**: Tipo del campo related. Tipo de relación para el campo related con el objeto de negocio.
* **relation**: Nombre del objeto de negocio al cual se aplica la relación con el campo tipo related.
* **string**: Texto para el campo tipo related.
* **store**: Este puede ser definida como True o False, segun lo requerido, con este valor estamos indicando si el valor devuelto por campo_busqueda es almacenado o no en la base de datos.

**Ejemplo de aplicación**:

	'titulo': fields.related(
		'libro_id','titulo',
		 type="char",
		 string="Titulo",
		 store=False,
		 readonly=True
	),

En este ejemplo se crea el campo titulo de tipo related el cual devuelve el titulo del libro según el valor del campo libro_id.

Domain
------

El atributo domain es un atributo aplicado a los campos para definir una restricción de dominio en un campo relacional, su valor por defecto es vacio.

**Estructura de definición**

	domain = [('campo', 'operador_comparacion', valor)])

* **campo**: nombre del campo del objeto de negocio, este campo será utilizado para la restricción.
* **operador_comparacion**: operador para comparar el campo con el valor establecido.
* **valor**: valor asignado para evaluar la restricción de dominio.

**Ejemplo de aplicación**:

Function
--------

Un campo funcional es un campo cuyo valor se calcula mediante una función, en lugar de ser almacenado en la base de datos.

**Estructura de definición**

	'nombre_campo_function' : fields.function(
            _nombre_metodo,
            type='tipo_campo',
            obj= "objeto_de_negocio.function",
            method= True,
            string= "Texto del campo"
	)

* **_nombre_metodo**: nombre del método creado para realizar el calculo.
* **type**: Tipo del campo funtion. Tipo de relación para el campo function con el objeto de negocio obj.
* **obj**: objeto de negocio al cual se aplica la relacion con el campo function.
* **method**: True.
* **string**: Texto para el campo tipo function.

**Ejemplo de aplicación**:

Ejercicios propuestos
---------------------

1. Revisar los cambios realizados en el módulo ejemplo biblioteca.
1. Adicionar un campo tipo function.
