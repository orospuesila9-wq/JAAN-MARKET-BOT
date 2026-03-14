import sqlite3
import logging
import random
import string
from datetime import datetime
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, LinkPreviewOptions
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, ContextTypes, MessageHandler, filters, Defaults

# --- AYARLAR ---
TOKEN = "8779032108:AAFDFzrOIv00-rt5-RLYuuvRJ_vjmaBNZAc"
OWNER_ID = 7735349550  
LOG_GRUP_ID = -1003627667857 

# --- KANAL LİSTESİ GÜNCELLENDİ ---
KANALLAR = ["@jaanyenikanal", "@JAANYEDEK3", "@JAANYEDEK2", "@prbotsupport"]

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# --- VERİTABANI YÖNETİMİ ---
def execute_query(query, params=(), is_read=False):
    conn = sqlite3.connect('bot_data.db', timeout=10)
    cursor = conn.cursor()
    try:
        cursor.execute(query, params)
        if is_read:
            return cursor.fetchall()
        conn.commit()
        return True
    except Exception as e:
        logging.error(f"DB Hatası: {e}")
        return [] if is_read else None
    finally:
        conn.close()

def db_setup():
    execute_query('''CREATE TABLE IF NOT EXISTS users (user_id INTEGER PRIMARY KEY, username TEXT, ref_count INTEGER DEFAULT 0, joined_at TEXT)''')
    execute_query('''CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, price INTEGER, stock INTEGER DEFAULT 0)''')
    execute_query('''CREATE TABLE IF NOT EXISTS admins (user_id INTEGER PRIMARY KEY)''')
    execute_query('''CREATE TABLE IF NOT EXISTS promo_codes (code TEXT PRIMARY KEY, reward INTEGER, is_used INTEGER DEFAULT 0)''')
    execute_query("INSERT OR IGNORE INTO admins (user_id) VALUES (?)", (OWNER_ID,))

# --- YARDIMCI FONKSİYONLAR ---
async def check_membership(context, user_id):
    for kanal in KANALLAR:
        try:
            m = await context.bot.get_chat_member(chat_id=kanal, user_id=user_id)
            if m.status in ['left', 'kicked']: return False
        except: return False
    return True

def is_admin(user_id):
    res = execute_query("SELECT user_id FROM admins WHERE user_id=?", (user_id,), is_read=True)
    return len(res) > 0

# --- ANA MENÜ ---
async def show_main_menu(update_or_query, context, user_id):
    is_member = await check_membership(context, user_id)
    imza = "\n\n👨‍💻 **YAPIMCI:** @JAANYENIDEN"
    u_data = execute_query("SELECT ref_count FROM users WHERE user_id=?", (user_id,), is_read=True)
    bal = u_data[0][0] if u_data else 0

    if not is_member:
        kb = [[InlineKeyboardButton(f"📢 Katıl: {k}", url=f"https://t.me/{k.replace('@','')}")] for k in KANALLAR]
        kb.append([InlineKeyboardButton("✅ Kontrol Et", callback_data="check_subs")])
        text = "⚠️ **Devam etmek için kanallara katılmalısın!**" + imza
    else:
        bot_obj = await context.bot.get_me()
        ref_link = f"https://t.me/{bot_obj.username}?start={user_id}"
        kb = [
            [InlineKeyboardButton("🛒 Mağaza", callback_data="market")],
            [InlineKeyboardButton("🎁 Kod Kullan", callback_data="use_code"), InlineKeyboardButton("🎫 Kod Üret", callback_data="gen_code")]
        ]
        if is_admin(user_id):
            kb.append([InlineKeyboardButton("⚙️ Admin Paneli", callback_data="admin_panel")])
        text = f"👋 **Hoş Geldin!**\n\n💰 Bakiyeniz: `{bal}` Ref\n🆔 ID: `{user_id}`\n\n🔗 **Referans Linkin:**\n`{ref_link}`" + imza

    rm = InlineKeyboardMarkup(kb)
    try:
        if isinstance(update_or_query, Update) and update_or_query.message:
            await update_or_query.message.reply_text(text=text, reply_markup=rm)
        else:
            await update_or_query.edit_message_text(text=text, reply_markup=rm)
    except: pass

# --- BUTON YÖNETİMİ ---
async def button_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    uid, data = query.from_user.id, query.data
    user = query.from_user
    await query.answer()

    if data == "market":
        prods = execute_query("SELECT id, name, price FROM products WHERE stock > 0", is_read=True)
        if not prods:
            await query.edit_message_text("🛒 Mağaza şu an boş!", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("⬅️ Geri", callback_data="back")]]))
            return
        kb = [[InlineKeyboardButton(f"🛍 {p[1]} | {p[2]} Ref", callback_data=f"buy_{p[0]}")] for p in prods]
        kb.append([InlineKeyboardButton("⬅️ Geri", callback_data="back")])
        await query.edit_message_text("🛒 **Mağaza Ürünleri**", reply_markup=InlineKeyboardMarkup(kb))

    elif data.startswith("buy_"):
        p_id = data.split("_")[1]
        prod = execute_query("SELECT name, price, stock FROM products WHERE id=?", (p_id,), is_read=True)
        if prod:
            p_name, p_price, p_stock = prod[0]
            u_data = execute_query("SELECT ref_count FROM users WHERE user_id=?", (uid,), is_read=True)
            u_bal = u_data[0][0] if u_data else 0
            if u_bal >= p_price:
                execute_query("UPDATE users SET ref_count = ref_count - ? WHERE user_id=?", (p_price, uid))
                execute_query("UPDATE products SET stock = stock - 1 WHERE id=?", (p_id,))
                await query.edit_message_text(f"✅ Ürün Alındı: `{p_name}`", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("⬅️ Geri", callback_data="back")]]))
                await context.bot.send_message(LOG_GRUP_ID, f"🛍 **SİPARİŞ!**\nAlıcı: @{user.username}\nÜrün: {p_name}")
            else: await query.answer("❌ Yetersiz bakiye!", show_alert=True)

    elif data == "admin_panel" and is_admin(uid):
        kb = [
            [InlineKeyboardButton("➕ Ürün Ekle", callback_data="admin_add_prod"), InlineKeyboardButton("➖ Ürün Çıkart", callback_data="admin_remove_prod_list")],
            [InlineKeyboardButton("🎫 Admin Kod Üret", callback_data="admin_gen_code")]
        ]
        if uid == OWNER_ID:
            kb.append([InlineKeyboardButton("👤 Admin Ekle", callback_data="admin_add_new"), InlineKeyboardButton("🚫 Admin Çıkar", callback_data="admin_remove_list")])
        kb.append([InlineKeyboardButton("⬅️ Geri", callback_data="back")])
        await query.edit_message_text("🛠 **Admin Paneli**", reply_markup=InlineKeyboardMarkup(kb))

    elif data == "admin_add_prod" and is_admin(uid):
        context.user_data["action"] = "waiting_new_product"
        await query.edit_message_text("📦 Format: `İsim, Fiyat, Stok`", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ İptal", callback_data="admin_panel")]]))

    elif data == "admin_gen_code" and is_admin(uid):
        context.user_data["action"] = "admin_gen_amt"
        await query.edit_message_text("🎫 Üretilecek Ref miktarı?", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ İptal", callback_data="admin_panel")]]))

    elif data == "admin_add_new" and uid == OWNER_ID:
        context.user_data["action"] = "waiting_new_admin_id"
        await query.edit_message_text("👤 Yeni Admin ID gönderin:", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ İptal", callback_data="admin_panel")]]))

    elif data == "admin_remove_list" and uid == OWNER_ID:
        admins = execute_query("SELECT user_id FROM admins", is_read=True)
        kb = [[InlineKeyboardButton(f"❌ Sil: {a[0]}", callback_data=f"rem_adm_{a[0]}")] for a in admins if a[0] != OWNER_ID]
        kb.append([InlineKeyboardButton("⬅️ Geri", callback_data="admin_panel")])
        await query.edit_message_text("🚫 Yetki Kaldır:", reply_markup=InlineKeyboardMarkup(kb))

    elif data.startswith("rem_adm_"):
        execute_query("DELETE FROM admins WHERE user_id=?", (data.split("_")[2],))
        await query.answer("✅ Yetki Alındı.")
        await button_handler(update, context)

    elif data == "admin_remove_prod_list" and is_admin(uid):
        prods = execute_query("SELECT id, name FROM products", is_read=True)
        kb = [[InlineKeyboardButton(f"🗑 Sil: {p[1]}", callback_data=f"del_prod_{p[0]}")] for p in prods]
        kb.append([InlineKeyboardButton("⬅️ Geri", callback_data="admin_panel")])
        await query.edit_message_text("📦 Ürün sil:", reply_markup=InlineKeyboardMarkup(kb))

    elif data.startswith("del_prod_"):
        execute_query("DELETE FROM products WHERE id=?", (data.split("_")[2],))
        await query.answer("✅ Silindi.")
        await button_handler(update, context)

    elif data == "check_subs":
        if await check_membership(context, uid): await show_main_menu(query, context, uid)
        else: await query.answer("❌ Katılmamışsın!", show_alert=True)

    elif data == "use_code":
        context.user_data["action"] = "waiting_promo_code"
        await query.edit_message_text("🎁 **Kodu gönderin:**", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ İptal", callback_data="back")]]))

    elif data == "gen_code":
        context.user_data["action"] = "waiting_gen_amount"
        await query.edit_message_text("🎫 **Tutar?**", reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("❌ İptal", callback_data="back")]]))

    elif data == "back":
        context.user_data.clear()
        await show_main_menu(query, context, uid)

# --- MESAJ İŞLEME ---
async def handle_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid, text = update.effective_user.id, update.message.text.strip()
    action = context.user_data.get("action")
    if not action: return

    if action == "waiting_promo_code":
        res = execute_query("SELECT reward, is_used FROM promo_codes WHERE code = ?", (text.upper(),), is_read=True)
        if res and res[0][1] == 0:
            execute_query("UPDATE promo_codes SET is_used = 1 WHERE code = ?", (text.upper(),))
            execute_query("UPDATE users SET ref_count = ref_count + ? WHERE user_id = ?", (res[0][0], uid))
            await update.message.reply_text(f"✅ +{res[0][0]} Ref eklendi.")
        else: await update.message.reply_text("❌ Geçersiz kod.")
    
    elif action == "waiting_gen_amount" and text.isdigit():
        amt = int(text)
        u_bal = execute_query("SELECT ref_count FROM users WHERE user_id=?", (uid,), is_read=True)
        if u_bal and u_bal[0][0] >= amt:
            code = "REF-" + ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
            execute_query("UPDATE users SET ref_count = ref_count - ? WHERE user_id=?", (amt, uid))
            execute_query("INSERT INTO promo_codes (code, reward, is_used) VALUES (?, ?, 0)", (code, amt))
            await update.message.reply_text(f"🎫 Kod: `{code}`")
        else: await update.message.reply_text("❌ Yetersiz bakiye.")

    elif action == "admin_gen_amt" and is_admin(uid) and text.isdigit():
        code = "ADM-" + ''.join(random.choices(string.ascii_uppercase + string.digits, k=10))
        execute_query("INSERT INTO promo_codes (code, reward, is_used) VALUES (?, ?, 0)", (code, int(text)))
        await update.message.reply_text(f"👑 Admin Kodu: `{code}`\nDeğer: {text} Ref")

    elif action == "waiting_new_admin_id" and uid == OWNER_ID and text.isdigit():
        execute_query("INSERT OR IGNORE INTO admins (user_id) VALUES (?)", (int(text),))
        await update.message.reply_text(f"✅ `{text}` admin olarak eklendi.")

    elif action == "waiting_new_product" and is_admin(uid):
        try:
            n, p, s = [i.strip() for i in text.split(",")]
            execute_query("INSERT INTO products (name, price, stock) VALUES (?, ?, ?)", (n, int(p), int(s)))
            await update.message.reply_text("✅ Ürün eklendi.")
        except: await update.message.reply_text("❌ Hata! Format: İsim, Fiyat, Stok")

    context.user_data.clear()
    await show_main_menu(update, context, uid)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid, username = update.effective_user.id, update.effective_user.username or f"User_{update.effective_user.id}"
    check = execute_query("SELECT user_id FROM users WHERE user_id=?", (uid,), is_read=True)
    if not check:
        ref_by = int(context.args[0]) if context.args and context.args[0].isdigit() else 0
        if ref_by and ref_by != uid:
            execute_query("UPDATE users SET ref_count = ref_count + 1 WHERE user_id=?", (ref_by,))
        execute_query("INSERT INTO users (user_id, username, joined_at) VALUES (?, ?, ?)", (uid, username, datetime.now().isoformat()))
    await show_main_menu(update, context, uid)

if __name__ == '__main__':
    db_setup()
    app = Application.builder().token(TOKEN).defaults(Defaults(parse_mode="Markdown", link_preview_options=LinkPreviewOptions(is_disabled=True))).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(button_handler))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_messages))
    print("🚀 Bot aktif! @prbotsupport listeye eklendi.")
    app.run_polling()
