<template>
  <div class="app-container">
    <Transition name="fade">
      <div v-if="mostrarNotificacion" class="notificacion-exito">
        ✅ Producto añadido al pedido
      </div>
    </Transition>

    <header class="main-header">
      <h1>Asistente gourmet</h1>
    </header>

    <div class="layout">
      <section class="catalogo">
        <div class="filter-bar">
          <label>Filtrar Menú:</label>
          <select v-model="categoriaSeleccionada" class="elegant-select">
            <option value="Todos">Ver Todo el Menú</option>
            <option v-for="cat in categorias" :key="cat" :value="cat">{{ cat }}</option>
          </select>
        </div>

        <div class="platos-grid">
          <template v-for="plato in platos" :key="plato.id">
            <div v-if="categoriaSeleccionada === 'Todos' || plato.categoria === categoriaSeleccionada" class="card">
              <div class="card-image">
                <img :src="plato.imagen" :alt="plato.nombre" />
                <span class="category-badge">{{ plato.categoria }}</span>
              </div>
              <div class="card-body">
                <h3>{{ plato.nombre }}</h3>
                <p class="description">{{ plato.descripcion }}</p>
                
                <div class="card-footer">
                  <div class="price-container">
                    <span class="price">${{ plato.precio.toLocaleString() }}</span>
                    <span :class="['stock-label', { 'agotado-text': plato.stock === 0 }]">
                      Disponibles: {{ plato.stock }}
                    </span>
                  </div>
                  <button 
                    class="btn-add-large" 
                    @click="agregar(plato)" 
                    :disabled="plato.stock === 0"
                  >
                    {{ plato.stock === 0 ? 'Agotado' : '＋ Agregar' }}
                  </button>
                </div>
              </div>
            </div>
          </template>
        </div>
      </section>

      <aside class="pedido-sidebar">
        <div class="ticket">
          <div class="ticket-header">
            <h2>Tu Pedido</h2>
          </div>
          
          <div v-if="pedido.length === 0" class="empty-cart">
            <p>Selecciona productos del menú</p>
          </div>

          <div v-else class="ticket-items">
            <div v-for="(p, i) in pedido" :key="i" class="ticket-row">
              <div class="item-info">
                <span class="item-name">{{ p.nombre }}</span>
                <span class="item-sub">${{ (p.precio * p.cantidad).toLocaleString() }}</span>
              </div>
              <div class="item-actions">
                <span class="qty-badge">x{{ p.cantidad }}</span>
                <button @click="eliminar(i, p)" class="btn-remove">❌</button>
              </div>
            </div>
          </div>

          <div class="ticket-footer">
            <div class="total-row">
              <span>Total:</span>
              <span class="total-price">${{ calcularTotal().toLocaleString() }}</span>
            </div>
            <button class="btn-pdf-premium" @click="generarFactura" :disabled="pedido.length === 0">
              <span class="icon">📄</span> FINALIZAR Y ABRIR FACTURA
            </button>
          </div>
        </div>
      </aside>
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import jsPDF from "jspdf";

const categorias = ["Comida Rápida", "Bebidas", "Postres", "Platos Fuertes"];
const categoriaSeleccionada = ref("Todos");
const pedido = ref([]);
const mostrarNotificacion = ref(false);

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
  { id: 22, nombre: "Sangría de Frutos Amarillos", categoria: "Bebidas", precio: 45000, stock: 20, descripcion: "Vino blanco con mango y maracuyá.", ingredientes: "Vino blanco, frutas.", imagen: "https://laveranerasangriaoficial.com/cdn/shop/files/WhatsAppImage2024-01-06at1.58.03PM_1.jpg?v=1751949367" },
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

const agregar = (plato) => {
  if (plato.stock > 0) {
    const item = pedido.value.find((p) => p.id === plato.id);
    if (item) {
      item.cantidad++;
    } else {
      pedido.value.push({ ...plato, cantidad: 1 });
    }
    plato.stock--;
    mostrarNotificacion.value = true;
    setTimeout(() => { mostrarNotificacion.value = false; }, 2000);
  }
};

const eliminar = (idx, itemPedido) => {
  const platoOriginal = platos.value.find(p => p.id === itemPedido.id);
  if (platoOriginal) { platoOriginal.stock += itemPedido.cantidad; }
  pedido.value.splice(idx, 1);
};

const calcularTotal = () => pedido.value.reduce((acc, p) => acc + (p.precio * p.cantidad), 0);

function generarFactura() {
  const doc = new jsPDF();
  const fecha = new Date().toLocaleString();
  
  doc.setFont("helvetica", "bold");
  doc.setFontSize(22);
  doc.text("RESTAURANTE GOURMET", 105, 20, { align: "center" });
  
  doc.setLineWidth(1);
  doc.line(20, 25, 190, 25); 

  doc.setFontSize(10);
  doc.setFont("helvetica", "normal");
  doc.text(`Fecha: ${fecha}`, 20, 35);
  doc.text("Nit: 900.555.222-1", 20, 40);

  let y = 60;
  doc.setFillColor(0, 0, 0);
  doc.rect(20, y - 5, 170, 8, 'F');
  doc.setTextColor(255, 255, 255);
  doc.text("PRODUCTO", 25, y);
  doc.text("CANT.", 110, y);
  doc.text("TOTAL", 170, y);

  doc.setTextColor(0, 0, 0);
  y += 10;
  pedido.value.forEach((p) => {
    doc.text(p.nombre.substring(0, 30), 25, y);
    doc.text(p.cantidad.toString(), 115, y, { align: "center" });
    doc.text(`$${(p.precio * p.cantidad).toLocaleString()}`, 170, y);
    y += 10;
  });

  doc.setLineWidth(0.5);
  doc.line(130, y, 190, y);
  doc.setFontSize(14);
  doc.setFont("helvetica", "bold");
  doc.text(`TOTAL: $${calcularTotal().toLocaleString()}`, 190, y + 10, { align: "right" });

  doc.output('dataurlnewwindow');
}
</script>

<style scoped>

.app-container { font-family: 'Segoe UI', Roboto, sans-serif; background: #f0f2f5; padding: 30px; min-height: 100vh; }
.main-header h1 { text-align: center; color: #1a1a1a; font-size: 2.5rem; margin-bottom: 40px; }

.layout { display: grid; grid-template-columns: 1fr 400px; gap: 30px; max-width: 1500px; margin: auto; }

.platos-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 25px; }
.card { background: white; border-radius: 20px; overflow: hidden; box-shadow: 0 10px 20px rgba(0,0,0,0.05); transition: 0.3s; }
.card:hover { transform: translateY(-5px); box-shadow: 0 15px 30px rgba(0,0,0,0.1); }
.card-image img { width: 100%; height: 200px; object-fit: cover; }
.card-body { padding: 20px; }

.btn-add-large { 
  background: #1a1a1a; 
  color: white; 
  border: none; 
  padding: 15px 25px; 
  border-radius: 12px; 
  font-weight: bold; 
  font-size: 1rem; 
  cursor: pointer; 
  transition: 0.2s; 
  width: 100%;
  margin-top: 10px;
}
.btn-add-large:hover:not(:disabled) { background: #fbbf24; color: #1a1a1a; transform: scale(1.02); }
.btn-add-large:disabled { background: #d1d5db; cursor: not-allowed; }

.ticket { background: white; border-radius: 25px; padding: 30px; position: sticky; top: 20px; border: 1px solid #e5e7eb; }
.btn-pdf-premium { 
  width: 100%; 
  padding: 20px; 
  background: #000; 
  color: #fff; 
  border: none; 
  border-radius: 15px; 
  font-size: 1.1rem; 
  font-weight: 800; 
  cursor: pointer; 
  letter-spacing: 1px;
  transition: 0.3s;
  box-shadow: 0 10px 20px rgba(0,0,0,0.2);
}
.btn-pdf-premium:hover:not(:disabled) { background: #333; transform: translateY(-3px); box-shadow: 0 15px 30px rgba(0,0,0,0.3); }
.btn-pdf-premium:disabled { background: #9ca3af; box-shadow: none; cursor: not-allowed; }

.price { font-size: 1.5rem; font-weight: 900; color: #1a1a1a; }
.stock-label { font-size: 0.85rem; font-weight: bold; color: #059669; }
.total-price { font-size: 2rem; font-weight: 900; color: #1a1a1a; }
.notificacion-exito { position: fixed; top: 30px; right: 30px; background: #10b981; color: white; padding: 20px 40px; border-radius: 15px; z-index: 999; font-weight: bold; }

.elegant-select { padding: 12px; border-radius: 10px; border: 2px solid #e5e7eb; width: 300px; font-weight: bold; }
</style>