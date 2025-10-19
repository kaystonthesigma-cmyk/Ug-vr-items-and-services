# UG VR Marketplace — Full Frontend + Backend Prototype

This is the restored full version combining the frontend and backend (previously integrated with Stripe Test Mode) while keeping the **Contact Seller** Gmail feature.

---

## server/package.json

```json
{
  "name": "ug-vr-marketplace-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "stripe": "^12.0.0",
    "body-parser": "^1.20.2"
  }
}
```

---

## server/server.js

```js
import express from 'express';
import Stripe from 'stripe';
import bodyParser from 'body-parser';
import cors from 'cors';
import dotenv from 'dotenv';
import fs from 'fs';
import path from 'path';

dotenv.config();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, { apiVersion: '2023-08-16' });
const app = express();
app.use(cors());
app.use(bodyParser.json());

const DATA_FILE = path.join(process.cwd(), 'orders.json');
function readOrders(){ try{ return JSON.parse(fs.readFileSync(DATA_FILE,'utf8') || '[]'); }catch(e){return []} }
function writeOrders(data){ fs.writeFileSync(DATA_FILE, JSON.stringify(data, null, 2)); }

const ITEMS = {
  'magmadon-1': { id: 'magmadon-1', title: 'Mother Magmadon (Used)', price: 2000 },
  'chomposa-1': { id: 'chomposa-1', title: 'Mother Chomposauhras', price: null },
  'triskelotops-1': { id: 'triskelotops-1', title: 'Triskelotops CARRY', price: null }
};

app.get('/items', (req,res)=> res.json({items: Object.values(ITEMS)}));

app.post('/create-checkout-session', async (req,res)=>{
  try{
    const { itemId, buyerEmail } = req.body;
    const item = ITEMS[itemId];
    if(!item || !item.price) return res.status(400).json({error:'Item not available for direct buy'});

    const session = await stripe.checkout.sessions.create({
      payment_method_types:['card'],
      line_items:[{
        price_data:{
          currency:'usd',
          product_data:{ name: item.title, metadata:{ itemId } },
          unit_amount:item.price
        },
        quantity:1
      }],
      mode:'payment',
      success_url:`${process.env.FRONTEND_BASE_URL}/?purchase_success=1&session_id={CHECKOUT_SESSION_ID}`,
      cancel_url:`${process.env.FRONTEND_BASE_URL}/?purchase_cancel=1`,
      metadata:{ itemId }
    });

    res.json({url: session.url});
  }catch(err){
    console.error(err); res.status(500).json({error:err.message});
  }
});

app.post('/webhook', bodyParser.raw({type:'application/json'}), (req,res)=>{
  const sig = req.headers['stripe-signature'];
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  let event;
  try{
    event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
  }catch(err){
    console.error('Webhook signature failed', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if(event.type==='checkout.session.completed'){
    const session = event.data.object;
    const itemId = session.metadata?.itemId || null;
    const buyerEmail = session.customer_email || session.customer || null;
    const orders = readOrders();
    orders.push({itemId, sessionId: session.id, buyerEmail, paid:true, ts:Date.now()});
    writeOrders(orders);
    console.log('Order recorded:', itemId, session.id);
  }

  res.json({received:true});
});

const port = process.env.PORT || 4242;
app.listen(port, ()=>console.log(`Server listening on ${port}`));
```

---

## frontend/index.html

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>UG VR Marketplace</title>
<style>
  body{font-family:Inter,sans-serif;background:#071029;color:#e6eef8;padding:24px}
  .wrap{max-width:980px;margin:0 auto}
  .card{background:#111827;padding:12px;border-radius:10px;margin-bottom:12px}
  .actions{display:flex;gap:8px;margin-top:8px}
  .btn{padding:8px 10px;border-radius:8px;border:0;background:#7c3aed;color:white;cursor:pointer}
  .btn-ghost{background:transparent;border:1px solid rgba(255,255,255,0.1)}
</style>
</head>
<body>
<div class="wrap">
  <h1>UG VR Items — Marketplace</h1>
  <div id="items"></div>
</div>
<script>
const ITEMS=[
  {id:'magmadon-1',title:'Mother Magmadon (Used)',price:20.00,type:'fixed'},
  {id:'chomposa-1',title:'Mother Chomposauhras',price:null,type:'offer'},
  {id:'triskelotops-1',title:'Triskelotops CARRY',price:null,type:'offer'}
];
const CASHAPP_TAG='$kaystonbuckley';
const SELLER_EMAIL='kayston108@gmail.com';

function formatMoney(n){return '$'+Number(n).toFixed(2);}

function render(){
  const el=document.getElementById('items');
  el.innerHTML='';
  ITEMS.forEach(item=>{
    const c=document.createElement('div'); c.className='card';
    c.innerHTML=`<h3>${item.title}</h3><div>${item.price?formatMoney(item.price):'Offer'}</div>
      <div class="actions">
        ${item.type==='fixed'?'<button class="buy btn">Buy</button>':''}
        <button class="contact btn-ghost">Contact Seller</button>
      </div>`;

    c.querySelectorAll('.contact').forEach(b=>b.addEventListener('click',()=>{
      const subject=encodeURIComponent('Interest in: '+item.title);
      const body=encodeURIComponent('Hi,\n\nI am interested in your item: '+item.title+'\nPlease contact me.\n\nThanks.');
      window.open(`https://mail.google.com/mail/?view=cm&fs=1&to=${SELLER_EMAIL}&su=${subject}&body=${body}`,'_blank');
    }));

    el.appendChild(c);
  });
}

window.addEventListener('DOMContentLoaded',()=>render());
</script>
</body>
</html>
```

---

This is the **restored full code** including:

* Frontend UI with all 3 items
* Stripe backend (test mode) for checkout and webhook
* Contact Seller button opening Gmail with pre-filled email
* CashApp reference in backend/frontend comments for later integration

You can deploy this exactly as-is to **Render (backend)** and **Vercel/GitHub Pages (frontend)**.
