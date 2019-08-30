Lección 08: Otros Widgets y Vistas
==================================

[TOC]

Widgets
-------

Widget es la clase base para todos los componentes visuales, un widget es un componente genérico dedicado a mostrar el contenido al usuario.

Los widgets son utilizados de manera automática de acuerdo al tipo de dato del campo a mostrar, pero estos widgets pueden ser manualmente seleccionados si es necesario. Existen diferentes widgets que pueden seleccionarse:

**widget="many2many_tags"**: Permite la selección para un campo Many2many al estilo de tags de la web 2.0
**widget="monetary"**: Permite visualizar el simbolo moneda, requiere que en el atributo `options` se indique el campo de tipo *res.currency* (tipo de moneda) a ser utilizado.
**widget="mail_followers"**: Permite adicionar seguidores a las notificaciones de un registro, el Modelo debe heredar de `mail.thread`
**widget="mail_thread"**: Despliega las notificaciones que tiene un registro, el Modelo debe heredar de `mail.thread`
**widget="statusbar"**: Muestra los valores de campo Selection en una barra, señalando el que esta actualmente seleccionado. En el widget se puede indicar por defecto cuales valores desplegar `statusbar_visible` y si es clickable para poder hacer cambiar la selección actual `clickable="1"`. Las opciones statusbar_visible y clickable son mutuamente excluyentes.
**widget="progressbar"**: Muestra el valor del campo en forma de una barra de progreso
**widget="url"**: Muestra el campo como un enlace web
**widget="integer"**: Permite almacenar solo valores enteros
**widget="image"**: Muestra el valor de un campo binary como una imagen
**widget="handle"**: Permite organizar un listado de registros y almacenar la posición en un campo con el nombre **sequence**. Para que los listados se desplieguen utilizando el campo sequences se debe adicionar en el Módelo el atributo _order="sequence ASC"
**widget="selection"**: Despliega un campo Many2one sin las opciones de buscar, crear o editar.

Ejemplo de aplicación de widget:

	<field name="state" widget="statusbar"/>
	<field name="presupuesto" widget="monetary" options="{'currency_field': 'currency_id'}"/>
    <field name="state" widget="statusbar"
        statusbar_visible="draft,sent,progress,invoiced,done"/>
    <field name="state" widget="statusbar" clickable="1"/>

Vista tipo kanban
-----------------

Las vistas tipo kanban permiten manejar información importante (campos principales) en imágenes o tarjetas, es util para poder identificar los registros de una forma más rápida y así agilizar la consulta sobre los datos.

Ejemplo para la creación de una vista tipo kanban:

	<record model="ir.ui.view" id="libro_kanban">
	<field name="name">libro.kanban</field>
	<field name="model">biblioteca.libro</field>
	<field name="type">kanban</field>
	<field name="arch" type="xml">
	<kanban version="7.0" class="oe_background_grey">
		<field name="titulo"/>
		<field name="autor"/>
		<field name="user_id"/>
		<templates>
			<t t-name="kanban-box">
				<div class="oe_resource_vignette">
					<div t-attf-class="oe_kanban_color_#{kanban_getcolor(record.state.value)} oe_kanban_card oe_kanban_global_click">
						<div class="oe_kanban_project_avatars">
							<img t-att-src="kanban_image('res.users', 'image', record.user_id.raw_value)"
								t-att-title="record.user_id.value"
								width="24" height="24"
								class="oe_kanban_avatar"
							/>
						</div>
					</div>
					<div class="oe_resource_details">
						<ul>
						   <li><field name="state"/></li>
						   <field name="titulo"/>
						   <field name="autor"/>
						 </ul>
					</div>
				</div>
			</t>
		</templates>
	</kanban>
	</field>
	</record>

Para que la vista se despliegue es necesario adicionar en el action_libro el tipo de vista kanban.

	<record model="ir.actions.act_window" id="action_libro">
	  <field name="name">Libro</field>
	  <field name="res_model">biblioteca.libro</field>
	  <field name="view_type">form</field>
	  <field name="view_mode">tree,form,graph,kanban</field>
	</record>


Vista tipo gantt
----------------

Las vistas tipo gantt permiten mostrar el tiempo planificado o transcurrido para el desarrollo de tareas o actividades, es una vista dinámica, se puede hacer click en cualquier parte del gráfico, arrastrar y soltar el objeto en otra ubicación.


Ejemplo para la creación de una vista tipo gantt:

	<record id="biblioteca_libro_prestamo_gantt" model="ir.ui.view">
		  <field name="name">biblioteca.libro_prestamo.gantt</field>
		  <field name="model">biblioteca.libro_prestamo</field>
		  <field eval="2" name="priority"/>
		  <field name="arch" type="xml">
			  <gantt date_start="fecha_prestamo" date_stop="fecha_regreso" string="Préstamos" default_group_by="libro_id">
			  </gantt>
		  </field>
	</record>

Para que la vista se despliegue es necesario adicionar en el action_libro_prestamo el tipo de vista gantt.

	<record model="ir.actions.act_window" id="action_libro_prestamo">
		  <field name="name">Prestamo</field>
		  <field name="res_model">biblioteca.libro_prestamo</field>
		  <field name="view_type">form</field>
		  <field name="view_mode">tree,form,gantt</field>
	</record>

Vista tipo calendar
-------------------

Las vistas tipo calendar permiten visualizar la planificación en tiempo para el desarrollo de tareas o actividades, es una vista dinámica, se puede hacer click en cualquier parte del gráfico, arrastrar y soltar el objeto en otra ubicación.

Ejemplo para la creación de una vista tipo calendar:

	<record id="biblioteca_libro_prestamo_calendar" model="ir.ui.view">
		  <field name="name">biblioteca.libro_prestamo.calendar</field>
		  <field name="model">biblioteca.libro_prestamo</field>
		  <field eval="2" name="priority"/>
		  <field name="arch" type="xml">
			  <calendar color="libro_id" date_start="fecha_prestamo" date_stop="fecha_regreso" string="Informe de Préstamos">
				  <field name="libro_id"/>
			  </calendar>
		  </field>
	</record>

Para que la vista se despliegue es necesario adicionar en el action_libro_prestamo el tipo de vista calendar.

	<record model="ir.actions.act_window" id="action_libro_prestamo">
		  <field name="name">Prestamo</field>
		  <field name="res_model">biblioteca.libro_prestamo</field>
		  <field name="view_type">form</field>
		  <field name="view_mode">tree,form,gantt,calendar</field>
	</record>


Vista tipo Gráfica
------------------

Esta vista permite desplegar los datos disponibles en una grafica. Cada vista gráfica puede ser definida utilizando los siguientes elementos básicos:

* atributo **`type`**: Indica el tipo de gráfica a utilizarse (pie, bar), por defecto pie
* atributo **`orientation`**: Indica la orientación de las barras (horizontal, vertical)
* **`field`**: Se debe ingresar como mínimo dos campos field (eje X, eje Y, eje Z), un tercero es opcional 3
* atributo **`group`**: Se coloca en 1 para el campo a ser utilizado en el GROUP BY
* atributo **`operator`**: Indica el tipo de operador de agregación a ser utilizado (+,*,**,min,max)

Ejemplo para la creación de una vista tipo gráfico:

    <record model="ir.ui.view" id="libro_graph">
        <field name="name">libro.graph</field>
        <field name="model">biblioteca.libro</field>
        <field name="arch" type="xml">
            <graph type="bar" orientation="horizontal" string="Gráfico">
                <field name="state"/>
                <field name="paginas" operator="min"/>
            </graph>
        </field>
    </record>

Botones
-------
- confirm
- states
- magic

                    <div class="oe_button_box oe_right">
                        <button name="button_fecha_compra_hoy" type="object" 
                            class="oe_inline oe_stat_button" icon="fa-calendar-o"
                            string="Comprado hoy"
                            confirm="Se marcará el libro como adquirido en la fecha de hoy, esta seguro?"
                            states="en_compra"
                        />
                    </div>

    @api.one
    def button_fecha_compra_hoy(self):
        self.fecha_compra = fields.Date.today()
        self.state = 'adquirido'


Ejercicios propuestos
---------------------

1. Verificar las diferentes vistas creadas.
1. Coloque el libro en estado *Proceso de compra* y haga click en el botón *Comprado hoy*, vea como cambian los campos *fecha de compra* y el *estado* del libro.
1. Adicioné un botón llamado *Devolver compra* que aparezca cuando el estado sea *Adquirido* y que cambie la fecha de compra a vacio y el estado a *Proceso de Compra*

