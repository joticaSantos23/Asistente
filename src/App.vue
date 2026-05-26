<template>
  <div class="app-container">
    <Transition name="slide-toast">
      <div v-if="notificacion.visible" :class="['notificacion-toast', notificacion.tipo]">
        {{ notificacion.mensaje }}
      </div>
    </Transition>

    <header class="main-header">
      <h1>✨ Restaurante Gourmet</h1>
      <p class="subtitle">Panel de Control & Gestión de Inventario</p>
    </header>

    <div class="layout-full">
      <section class="catalogo">
        <div class="toolbar">
          <div class="filter-group">
            <label> Categoría:</label>
            <select v-model="categoriaSeleccionada" class="elegant-select">
              <option value="Todos"> Mostrar Todo</option>
              <option v-for="cat in categorias" :key="cat" :value="cat">{{ cat }}</option>
            </select>
          </div>
          <button class="btn-add-product" @click="abrirModalNuevoPlato">
            + Nuevo Producto
          </button>
        </div>

        <div class="platos-grid">
          <template v-for="(plato, index) in platos" :key="plato.id">
            <div v-if="categoriaSeleccionada === 'Todos' || plato.categoria === categoriaSeleccionada" class="card">
              <div class="card-image">
                <img :src="plato.imagen" :alt="plato.nombre" />
                <span class="category-badge">{{ plato.categoria }}</span>
                <button class="btn-delete-from-inventory" @click="eliminarDelInventario(plato.id)" title="Eliminar producto del menú">
                  🗑️
                </button>
              </div>
              <div class="card-body">
                <h3>{{ plato.nombre }}</h3>
                <p class="description">{{ plato.descripcion }}</p>
                
                <div class="card-footer">
                  <div class="price-container">
                    <span class="price">${{ plato.precio.toLocaleString() }}</span>
                    <span :class="['stock-label', { 'agotado-text': plato.stock === 0 }]">
                      Stock: {{ plato.stock }}
                    </span>
                  </div>
                  <button class="btn-add-to-cart" @click="agregarAlPedido(plato)" :disabled="plato.stock === 0">
                    {{ plato.stock === 0 ? 'Agotado' : '＋ Agregar' }}
                  </button>
                </div>
              </div>
            </div>
          </template>
        </div>
      </section>
    </div>

    <button class="btn-floating-cart" @click="carritoAbierto = true">
      <span class="cart-icon">🛒</span>
      <span v-if="pedido.length > 0" class="cart-badge-count">{{ obtenerCantidadTotal() }}</span>
    </button>

    <Transition name="fade">
      <div v-if="carritoAbierto" class="modal-overlay" @click="carritoAbierto = false">
        <div class="modal-content ticket-modal" @click.stop>
          <div class="ticket-header">
            <h2>🧾 Resumen de Pedido</h2>
            <button @click="carritoAbierto = false" class="btn-close-circle">✕</button>
          </div>
          
          <div v-if="pedido.length === 0" class="empty-cart">
            <p>Carrito vacío</p>
          </div>

          <div v-else class="ticket-items">
            <div v-for="(p, i) in pedido" :key="i" class="ticket-row">
              <div class="item-info">
                <span class="item-name">{{ p.nombre }}</span>
                <small>${{ p.precio.toLocaleString() }} c/u</small>
              </div>
              <div class="item-actions">
                <span class="qty-badge">x{{ p.cantidad }}</span>
                <button @click="eliminarUno(i, p)" class="btn-remove-black">✕</button>
              </div>
            </div>
          </div>

          <div class="ticket-footer">
            <div class="total-row">
              <span>Total:</span>
              <span class="total-price">${{ calcularTotal().toLocaleString() }}</span>
            </div>
            <button class="btn-finalize" @click="procesarPago" :disabled="pedido.length === 0 || cargando">
              <span v-if="cargando" class="spinner"></span>
              <span v-else>💳 FINALIZAR Y ABRIR FACTURA</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>

    <Transition name="fade">
      <div v-if="mostrarModalNuevo" class="modal-overlay" @click="mostrarModalNuevo = false">
        <div class="modal-content form-modal" @click.stop>
          <div class="ticket-header">
            <h2>🍱 Agregar Nuevo Plato</h2>
            <button @click="mostrarModalNuevo = false" class="btn-close-circle">✕</button>
          </div>
          
          <form @submit.prevent="guardarNuevoPlato" class="nuevo-plato-form">
            <div class="form-group">
              <label>Nombre del Plato:</label>
              <input v-model="nuevoPlato.nombre" type="text" required placeholder="Ej: Pizza Hawaiana" />
            </div>

            <div class="form-row">
              <div class="form-group">
                <label>Categoría:</label>
                <select v-model="nuevoPlato.categoria">
                  <option v-for="cat in categorias" :key="cat" :value="cat">{{ cat }}</option>
                </select>
              </div>
              <div class="form-group">
                <label>Stock Inicial:</label>
                <input v-model.number="nuevoPlato.stock" type="number" min="1" required />
              </div>
            </div>

            <div class="form-group">
              <label>Precio (COP):</label>
              <input v-model.number="nuevoPlato.precio" type="number" min="0" required />
            </div>

            <div class="form-group">
              <label>Imagen del Producto (.jpg):</label>
              <input type="file" @change="manejarImagen" accept="image/jpeg, image/jpg" required />
            </div>

            <button type="submit" class="btn-save-product">💾 GUARDAR EN MENÚ</button>
          </form>
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref } from "vue";
import jsPDF from "jspdf";

const categorias = ["Comida Rápida", "Bebidas", "Postres", "Platos Fuertes"];
const categoriaSeleccionada = ref("Todos");
const pedido = ref([]);
const carritoAbierto = ref(false);
const mostrarModalNuevo = ref(false);
const cargando = ref(false);
const notificacion = ref({ visible: false, mensaje: "", tipo: "success" });

const platos = ref([
  { id: 1, nombre: "Hamburguesa Premium", categoria: "Comida Rápida", precio: 15000, stock: 20, descripcion: "Carne angus con queso fundido.", ingredientes: "Pan brioche, carne.", imagen: "https://amebeef.com/wp-content/uploads/2024/03/american-beef-carne-para-hamburguesa-premium-beef-5.jpg" },
  { id: 2, nombre: "Pizza Pepperoni", categoria: "Comida Rápida", precio: 22000, stock: 20, descripcion: "Masa delgada y crujiente.", ingredientes: "Queso, pepperoni.", imagen: "https://www.hormel.com/brands/hormel-pepperoni/wp-content/uploads/sites/3/Recipes_2400_Spicy-Pepperoni-Pizza.jpg?1776795456" },
  { id: 3, nombre: "Coca Cola", categoria: "Bebidas", precio: 5000, stock: 20, descripcion: "Bebida refrescante.", ingredientes: "Hielo.", imagen: "https://thumbs.dreamstime.com/b/ca%C3%B1a-y-vidrio-de-coca-cola-con-hielo-poznan-pol-nov-can-una-bebida-gaseosa-carbonatada-fabricada-por-company-sede-en-atlanta-164904056.jpg" },
  { id: 4, nombre: "Brownie", categoria: "Postres", precio: 12000, stock: 20, descripcion: "Chocolate caliente.", ingredientes: "Helado.", imagen: "https://kitchenfair.com.mx/wp-content/uploads/2017/07/receta_brownies_1280x854.jpg" },
  { id: 5, nombre: "Filete Miñón", categoria: "Platos Fuertes", precio: 45000, stock: 20, descripcion: "Corte tierno.", ingredientes: "Salsa champiñones.", imagen: "https://peopleenespanol.com/thmb/Qzlz7HhLeN1KdJLdltoSzdVSLH4=/1500x0/filters:no_upscale():max_bytes(150000):strip_icc()/recetas-1073-pollo-con-champinones-a-la-crema-2000-994f97be27a8468180f6985119917802.jpg" },
  { id: 6, nombre: "Filete Mignon al Merlot", categoria: "Platos Fuertes", precio: 85000, stock: 20, descripcion: "Corazón de solomillo envuelto en tocino ahumado.", ingredientes: "Solomillo, tocino, vino Merlot.", imagen: "https://adamsfarms.com/wp-content/uploads/2013/12/Bacon-Wrapped-Beef-Tenderloin.jpg" },
  { id: 7, nombre: "Salmón en Costra de Eneldo", categoria: "Platos Fuertes", precio: 72000, stock: 20, descripcion: "Filete fresco con puré de coliflor trufado.", ingredientes: "Salmón, eneldo, coliflor.", imagen: "https://cdn.shopify.com/s/files/1/0497/1659/2805/files/paso_5.jpg?v=1753276967" },
  { id: 8, nombre: "Risotto de Hongos Silvestres", categoria: "Platos Fuertes", precio: 58000, stock: 20, descripcion: "Arroz cremoso con lascas de queso Grana Padano.", ingredientes: "Arroz arbóreo, setas, queso.", imagen: "https://i.blogs.es/38ad9a/risotto_setas/450_1000.jpg" },
  { id: 9, nombre: "Langosta a la Thermidor", categoria: "Platos Fuertes", precio: 120000, stock: 20, descripcion: "Cola de langosta gratinada con crema de coñac.", ingredientes: "Langosta, coñac, mostaza.", imagen: "https://comerbeber.com/archivos/styles/max_900x600/public/imagen/2020/12/xlangosta-gratinada-cv_1200.jpg,qitok=9_ONafzh.pagespeed.ic.oifyDuXt_f.jpg" },
  { id: 11, nombre: "Pato al Naranja y Cointreau", categoria: "Platos Fuertes", precio: 78000, stock: 20, descripcion: "Pechuga de pato con vegetales baby glaseados.", ingredientes: "Pato, naranja, licor.", imagen: "https://wwv.sherry.wine/media/images/duck_breasts_tarragon_orange_glaze_amontillado.width-876.jpg" },
  { id: 12, nombre: "Cordero en Costra de Pistacho", categoria: "Platos Fuertes", precio: 82000, stock: 20, descripcion: "Costillas premium con puré de guisantes.", ingredientes: "Cordero, pistacho, menta.", imagen: "https://www.chefcaprabo.com/export/shared/.galleries/recetas/20_5201332CAS_682x433.jpg_908605617.jpg" },
  { id: 13, nombre: "Raviolis de Langostinos", categoria: "Platos Fuertes", precio: 55000, stock: 20, descripcion: "Pasta artesanal en salsa de azafrán.", ingredientes: "Pasta, langostinos, azafrán.", imagen: "https://i.blogs.es/0f9cd7/espaguettis_azafran/650_1200.jpg" },
  { id: 14, nombre: "Volcán de Chocolate Amargo", categoria: "Postres", precio: 28000, stock: 20, descripcion: "Bizcocho fundido con helado de vainilla.", ingredientes: "Cacao 70%, vainilla.", imagen: "https://cadiz.cosasdecome.es/wp-content/uploads/2017/09/41-Bizcocho-templado-con-helado-de-vainilla-y-chocolate-caliente-en-Burlesque-de-C%C3%A1diz.jpg" },
  { id: 15, nombre: "Crème Brûlée de Lavanda", categoria: "Postres", precio: 24000, stock: 20, descripcion: "Crema infusionada con costra de caramelo.", ingredientes: "Nata, lavanda, azúcar.", imagen: "https://thumbs.dreamstime.com/b/cr%C3%A8me-br%C3%BBl%C3%A9e-postre-con-costra-de-caramelo-elegante-gourmet-az%C3%BAcar-caramelizada-crema-vainilla-francesa-comer-cuchara-romper-414779477.jpg" },
  { id: 16, nombre: "Tarta de Queso y Frutos Rojos", categoria: "Postres", precio: 26000, stock: 20, descripcion: "Base artesanal con mousse de queso premium.", ingredientes: "Queso crema, frambuesas.", imagen: "https://bsstatic.mrjack.es/wp-content/uploads/flickr/a6c/b25/7189003825_d0f9f52e06_b.jpg" },
  { id: 17, nombre: "Milhojas de Crema Diplomata", categoria: "Postres", precio: 22000, stock: 20, descripcion: "Hojaldre crujiente con lluvia de pistachos.", ingredientes: "Hojaldre, crema, pistacho.", imagen: "https://junaenlacocina.com/wp-content/uploads/2026/01/hojaldre-crema-pistacho-y-pepitas-chocolate-1024x902.jpg" },
  { id: 18, nombre: "Mousse de Maracuyá , Coco y Mango", categoria: "Postres", precio: 20000, stock: 20, descripcion: "Equilibrio entre acidez y cremosidad.", ingredientes: "Maracuyá, coco, crema, mango.", imagen: "https://www.revistapancaliente.co/wp-content/uploads/2024/07/COCO-CHOCOLATE-1.webp" },
  { id: 19, nombre: "Peras al Vino Tinto", categoria: "Postres", precio: 25000, stock: 20, descripcion: "Peras pochadas con helado de jengibre.", ingredientes: "Peras, vino, jengibre.", imagen: "https://cdn.blog.paulinacocina.net/wp-content/uploads/2024/04/receta-de-peras-al-vino-tinto-Paulina-Cocina-Recetas-Cocina-1722430350.jpg" },
  { id: 20, nombre: "Affogato Imperial", categoria: "Postres", precio: 18000, stock: 20, descripcion: "Gelatto de avellana con espresso y Amaretto.", ingredientes: "Gelatto, caffé, licor.", imagen: "https://shop.raynvillesuperstore.co.uk/cdn/shop/files/affogato-imperial-stout-bourbon-barrel-aged-107-330ml-bottle-849243_1024x1024_2x_1_1200x1200.webp?v=1698239639" },
  { id: 21, nombre: "Limonada de Coco y Menta", categoria: "Bebidas", precio: 15000, stock: 20, descripcion: "Mezcla refrescante de coco y limones.", ingredientes: "Leche de coco, limón.", imagen: "https://www.recetasnestle.com.co/sites/default/files/srh_recipes/422ebc949f85f0119fe5ca3beff55a17.jpg" },
  { id: 22, nombre: "Sangría de Frutos Rojos", categoria: "Bebidas", precio: 45000, stock: 20, descripcion: "Vino tinto con distintas frutas.", ingredientes: "Vino tinto, frutas.", imagen: "https://alimentossnob.com/wp-content/uploads/2020/10/sangriamedio-scaled.jpg" },
  { id: 23, nombre: "Gin Tonic de Pepino y Rosas", categoria: "Bebidas", precio: 38000, stock: 20, descripcion: "Ginebra premium con pétalos orgánicos.", ingredientes: "Gin, tónica, pepino.", imagen: "https://cuk-it.com/wp-content/uploads/2024/12/gin-tonic-pepino.webp" },
  { id: 24, nombre: "Vino Tinto Reserva (Copa)", categoria: "Bebidas", precio: 32000, stock: 20, descripcion: "Selección de la casa Malbec o Cabernet.", ingredientes: "Uva reserva.", imagen: "https://lacaretalicores.com/cdn/shop/files/VINO_CASTILLO_MOLINA.webp?v=1764554804&width=600" },
  { id: 25, nombre: "Infusión de Frutos Rojos", categoria: "Bebidas", precio: 12000, stock: 20, descripcion: "Mezcla de bayas y flores de hibisco.", ingredientes: "Bayas, hibisco.", imagen: "https://www.pompadour.es/modules/ph_simpleblog/featured/141.jpg" },
  { id: 26, nombre: "Martini de Espresso", categoria: "Bebidas", precio: 35000, stock: 20, descripcion: "Vodka con café artesanal recién tostado.", ingredientes: "Vodka, café, licor.", imagen: "https://www.vinoskichak.com/cdn/shop/articles/ESPRESSO_MARTINI.jpg?v=1708916663" },
  { id: 27, nombre: "Agua Mineral Premium", categoria: "Bebidas", precio: 10000, stock: 20, descripcion: "Agua de manantial con cítricos y romero.", ingredientes: "Agua, limón, romero.", imagen: "https://resizer.glanacion.com/resizer/v2/JMRDGR6W2FHRNMJB62JMNH3MJQ.jpg?auth=e437a950b558c5efc6d4d331ebfcf0092b05b2423bb8c25139e82e5713b2c2eb&width=420&height=236&quality=70&smart=true" },
  { id: 28, nombre: "Pizza de Trufa y Prosciutto", categoria: "Comida Rápida", precio: 48000, stock: 20, descripcion: "Aceite de trufa blanca y jamón serrano.", ingredientes: "Trufa, prosciutto.", imagen: "https://www.giallozafferano.es/images/151-15155/pizza-con-crema-de-trufa-y-jamon_1200x800.jpg" },
  { id: 29, nombre: "Tacos de Atún Sellado", categoria: "Comida Rápida", precio: 42000, stock: 20, descripcion: "Tortillas con atún fresco y aguacate.", ingredientes: "Atún, aguacate.", imagen: "https://cdn7.kiwilimon.com/recetaimagen/29728/960x640/31324.jpg.jpg" },
  { id: 30, nombre: "Sandwich de Brisket Ahumado", categoria: "Comida Rápida", precio: 38000, stock: 20, descripcion: "Pecho de res ahumado por 16 horas.", ingredientes: "Brisket, masa madre.", imagen: "https://chili-inc.com/cdn/shop/articles/WhatsApp_Image_2021-11-29_at_18.05.09.jpg?v=1638227888" },
  { id: 31, nombre: "Papas Trufadas con Parmesano", categoria: "Comida Rápida", precio: 22000, stock: 20, descripcion: "Papa rústica con sal de mar y aceite de trufa.", ingredientes: "Papa, trufa, parmesano.", imagen: "https://sharkninjacolombia.com/cdn/shop/articles/Papas_fritas_con_salsa_de_trufa_y_parmesano.jpg?v=1732827683" },
  { id: 32, nombre: "Hot Dog de Salchicha Alemana", categoria: "Comida Rápida", precio: 34000, stock: 20, descripcion: "Pan brioche con chucrut de la casa.", ingredientes: "Salchicha, brioche.", imagen: "https://www.tangamanga.com/storage/app/uploads/public/62f/58e/f84/thumb_169_800_600_0_0_auto.png" },
  { id: 33, nombre: "Alitas al Glaseado de Balsámico", categoria: "Comida Rápida", precio: 36000, stock: 20, descripcion: "Alas bañadas en reducción de balsámico.", ingredientes: "Pollo, balsámico.", imagen: "https://www.homecookingadventure.com/wp-content/uploads/2018/03/Oven-BBQ-Chicken-Wings-main1.webp" },
  { id: 34, nombre: "Bowl de Quinoa y Salmón", categoria: "Comida Rápida", precio: 45000, stock: 20, descripcion: "Opción saludable con aderezo de tahini.", ingredientes: "Quinoa, salmón.", imagen: "https://www.brillante.es/wp-content/uploads/2022/11/Buddha-bowl-de-salmon-ahumado-y-arroz-integral-con-quinoa.jpg" }
]);

const nuevoPlato = ref({ nombre: "", categoria: "Comida Rápida", precio: 0, stock: 10, imagen: "" });

const abrirModalNuevoPlato = () => {
  nuevoPlato.value = { nombre: "", categoria: "Comida Rápida", precio: 0, stock: 10, imagen: "" };
  mostrarModalNuevo.value = true;
};

const manejarImagen = (e) => {
  const file = e.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (f) => { nuevoPlato.value.imagen = f.target.result; };
    reader.readAsDataURL(file);
  }
};

const guardarNuevoPlato = () => {
  const platoParaGuardar = { 
    ...nuevoPlato.value, 
    id: Date.now(), 
    descripcion: "Producto agregado recientemente al sistema." 
  };
  platos.value.push(platoParaGuardar);
  mostrarModalNuevo.value = false;
  lanzarAviso("✅ Producto añadido al catálogo");
};

// FUNCIÓN PARA BORRAR PRODUCTO POR COMPLETO
const eliminarDelInventario = (id) => {
  // 1. Quitarlo del catálogo general
  platos.value = platos.value.filter(p => p.id !== id);
  // 2. Si estaba en el carrito actual, limpiarlo de allí también
  pedido.value = pedido.value.filter(p => p.id !== id);
  lanzarAviso("🗑️ Producto eliminado del catálogo", "alert");
};

const agregarAlPedido = (plato) => {
  if (plato.stock > 0) {
    const existe = pedido.value.find(p => p.id === plato.id);
    if (existe) { existe.cantidad++; } 
    else { pedido.value.push({ ...plato, cantidad: 1 }); }
    plato.stock--;
    lanzarAviso("✨ Producto añadido");
  }
};

const eliminarUno = (index, item) => {
  const original = platos.value.find(p => p.id === item.id);
  if (original) original.stock++;
  if (item.cantidad > 1) { item.cantidad--; } 
  else { pedido.value.splice(index, 1); }
};

const calcularTotal = () => {
  let t = 0;
  pedido.value.forEach(p => t += (p.precio * p.cantidad));
  return t;
};

const obtenerCantidadTotal = () => {
  let c = 0;
  pedido.value.forEach(p => c += p.cantidad);
  return c;
};

const lanzarAviso = (msg, tipo = "success") => {
  notificacion.value = { visible: true, mensaje: msg, tipo };
  setTimeout(() => notificacion.value.visible = false, 2000);
};

const procesarPago = () => {
  cargando.value = true;
  setTimeout(() => {
    generarFactura();
    pedido.value = [];
    cargando.value = false;
    carritoAbierto.value = false;
  }, 1200);
};

// DISEÑO DEL PDF CORREGIDO (MÁXIMA SEPARACIÓN DE CAMPOS)
function generarFactura() {
  const doc = new jsPDF();
  const fecha = new Date().toLocaleString();
  const totalPagado = `$${calcularTotal().toLocaleString()}`;

  doc.setFont("helvetica", "bold");
  doc.setFontSize(22);
  doc.text("RESTAURANTE GOURMET", 105, 20, { align: "center" });
  doc.setLineWidth(1);
  doc.line(15, 25, 195, 25);

  doc.setFontSize(10);
  doc.setFont("helvetica", "normal");
  doc.text(`Fecha: ${fecha}`, 15, 35);
  doc.text("Nit: 900.555.222-1 | Régimen Simplificado", 15, 41);

  let y = 60;
  doc.setFillColor(0, 0, 0);
  doc.rect(15, y - 6, 180, 10, 'F');
  doc.setTextColor(255, 255, 255);
  doc.text("DETALLE DEL PRODUCTO", 20, y);
  doc.text("CANT.", 120, y);
  doc.text("SUBTOTAL", 185, y, { align: "right" });

  doc.setTextColor(0, 0, 0);
  y += 10;
  pedido.value.forEach(p => {
    doc.text(p.nombre, 20, y);
    doc.text(p.cantidad.toString(), 123, y);
    doc.text(`$${(p.precio * p.cantidad).toLocaleString()}`, 185, y, { align: "right" });
    y += 10;
  });

  y += 10;
  doc.setLineWidth(0.5);
  doc.line(15, y, 195, y); // Línea completa para armonía visual
  
  y += 12;
  doc.setFontSize(14);
  doc.setFont("helvetica", "bold");
  
  // SOLUCIÓN AL SOLAPAMIENTO: Anclados a extremos opuestos de la página
  doc.text("TOTAL NETO A PAGAR:", 20, y); 
  doc.text(totalPagado, 185, y, { align: "right" }); 

  doc.output('dataurlnewwindow');
}
</script>

<style scoped>
/* GENERAL */
.app-container { font-family: 'Inter', sans-serif; background: #f8fafc; min-height: 100vh; padding: 20px; position: relative; }
.main-header { text-align: center; margin-bottom: 25px; }
.main-header h1 { font-size: 2.2rem; font-weight: 800; color: #0f172a; margin: 0; }
.subtitle { color: #64748b; font-size: 0.9rem; }

/* TOOLBAR */
.toolbar { display: flex; justify-content: space-between; align-items: center; background: white; padding: 12px 20px; border-radius: 12px; margin-bottom: 20px; border: 1px solid #e2e8f0; }
.elegant-select { padding: 8px; border-radius: 8px; border: 1px solid #cbd5e1; font-weight: 600; }
.btn-add-product { background: #000; color: white; border: none; padding: 10px 18px; border-radius: 8px; font-weight: 700; cursor: pointer; transition: 0.2s; }
.btn-add-product:hover { background: #f59e0b; color: #000; }

/* REJILLA Y CARDS OPTIMIZADAS (SIN ESPACIO MUERTO) */
.platos-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 18px; }
.card { background: white; border-radius: 16px; overflow: hidden; border: 1px solid #e2e8f0; transition: 0.3s; display: flex; flex-direction: column; height: 100%; }
.card:hover { transform: translateY(-4px); box-shadow: 0 10px 20px rgba(0,0,0,0.06); }

.card-image { height: 160px; position: relative; background: #f1f5f9; }
.card-image img { width: 100%; height: 100%; object-fit: cover; }

.category-badge { position: absolute; bottom: 10px; right: 10px; background: rgba(0,0,0,0.75); color: #fbbf24; padding: 4px 8px; border-radius: 12px; font-size: 0.65rem; font-weight: 700; }

/* BOTÓN BORRAR DEL CATÁLOGO */
.btn-delete-from-inventory { position: absolute; top: 10px; left: 10px; background: rgba(255, 255, 255, 0.9); border: none; width: 28px; height: 28px; border-radius: 50%; display: flex; align-items: center; justify-content: center; cursor: pointer; box-shadow: 0 2px 5px rgba(0,0,0,0.15); transition: 0.2s; font-size: 0.85rem; }
.btn-delete-from-inventory:hover { background: #ef4444; transform: scale(1.1); }

/* CUERPO COMPACTO */
.card-body { padding: 12px; flex-grow: 1; display: flex; flex-direction: column; justify-content: space-between; gap: 6px; }
.card-body h3 { margin: 0; font-size: 1.05rem; font-weight: 700; color: #0f172a; }
.description { font-size: 0.8rem; color: #64748b; margin: 0; line-height: 1.3; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }

.card-footer { display: flex; justify-content: space-between; align-items: center; margin-top: 4px; padding-top: 8px; border-top: 1px solid #f1f5f9; }
.price { font-size: 1.15rem; font-weight: 800; color: #0f172a; }
.stock-label { font-size: 0.7rem; color: #059669; font-weight: 700; display: block; margin-top: 1px; }
.agotado-text { color: #dc2626; }

.btn-add-to-cart { background: #0f172a; color: white; border: none; padding: 6px 12px; border-radius: 6px; font-size: 0.85rem; font-weight: 700; cursor: pointer; transition: 0.2s; }
.btn-add-to-cart:hover { background: #f59e0b; color: black; }
.btn-add-to-cart:disabled { background: #e2e8f0; color: #94a3b8; cursor: not-allowed; }

/* MODALES Y CARRITO */
.btn-floating-cart { position: fixed; bottom: 25px; right: 25px; background: #000; color: white; width: 55px; height: 55px; border-radius: 50%; border: none; cursor: pointer; font-size: 1.4rem; z-index: 100; box-shadow: 0 4px 15px rgba(0,0,0,0.2); }
.cart-badge-count { position: absolute; top: -2px; right: -2px; background: #ef4444; color: white; font-size: 0.65rem; padding: 2px 6px; border-radius: 10px; font-weight: 800; }

.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(15, 23, 42, 0.4); backdrop-filter: blur(4px); display: flex; align-items: center; justify-content: center; z-index: 1000; }
.modal-content { background: white; border-radius: 20px; padding: 20px; width: 90%; max-width: 460px; max-height: 85vh; overflow-y: auto; box-shadow: 0 20px 25px -5px rgba(0,0,0,0.1); }
.ticket-header { display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #f1f5f9; padding-bottom: 8px; margin-bottom: 12px; }
.btn-close-circle { background: #f1f5f9; border: none; width: 28px; height: 28px; border-radius: 50%; cursor: pointer; font-weight: bold; }

/* FORMULARIOS */
.nuevo-plato-form { display: flex; flex-direction: column; gap: 12px; }
.form-group { display: flex; flex-direction: column; gap: 4px; }
.form-group label { font-weight: 700; font-size: 0.85rem; color: #334155; }
.form-group input, .form-group select { padding: 8px 12px; border-radius: 8px; border: 1px solid #e2e8f0; font-size: 0.9rem; }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.btn-save-product { background: #000; color: white; border: none; padding: 12px; border-radius: 10px; font-weight: 800; cursor: pointer; margin-top: 5px; }

/* TICKETS Y ELEMENTOS EN LISTA */
.ticket-items { display: flex; flex-direction: column; gap: 8px; }
.ticket-row { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px dashed #e2e8f0; padding-bottom: 6px; }
.item-name { font-weight: 700; font-size: 0.9rem; }
.btn-remove-black { background: #000; color: white; border: none; width: 20px; height: 20px; border-radius: 50%; cursor: pointer; font-size: 0.65rem; }
.total-row { display: flex; justify-content: space-between; align-items: center; margin: 15px 0; border-top: 2px solid #000; padding-top: 8px; }
.total-price { font-size: 1.6rem; font-weight: 900; }
.btn-finalize { width: 100%; background: #000; color: white; border: none; padding: 14px; border-radius: 12px; font-weight: 800; cursor: pointer; display: flex; justify-content: center; align-items: center; }

/* TRANSICIONES Y NOTIFICACIONES */
.spinner { width: 18px; height: 18px; border: 2px solid rgba(255,255,255,0.3); border-top-color: #fff; border-radius: 50%; animation: spin 0.8s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }
.slide-toast-enter-active, .slide-toast-leave-active { transition: 0.25s; }
.slide-toast-enter-from, .slide-toast-leave-to { transform: translateX(110%); opacity: 0; }
.notificacion-toast { position: fixed; top: 15px; right: 15px; background: #0f172a; color: #fbbf24; padding: 12px 20px; border-radius: 10px; z-index: 2000; font-weight: 700; font-size: 0.9rem; }
.notificacion-toast.alert { color: #f87171; }
.fade-enter-active, .fade-leave-active { transition: 0.2s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
