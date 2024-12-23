# Dev.py
from telegram import Update
from telegram.ext import Updater, CommandHandler, CallbackContext
from pymongo import MongoClient

MONGO_URI = "mongodb+srv://<username>:<password>@cluster.mongodb.net/postdb?retryWrites=true&w=majority"
client = MongoClient(MONGO_URI)
db = client["postdb"]
movies_collection = db["movies"]
requests_collection = db["requests"]

ADMIN_ID = 123456789

def start(update: Update, context: CallbackContext):
    update.message.reply_text(
        "Commands:\n/connections\n/findmovie <title>\nAdmin Commands:\n/approve <user_id>\n/reject <user_id>"
    )

def connections(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    user_name = update.message.from_user.username or "Unknown"
    requests_collection.insert_one({"user_id": user_id, "user_name": user_name, "status": "pending"})
    context.bot.send_message(chat_id=ADMIN_ID, text=f"New request:\nUser ID: {user_id}\nUsername: @{user_name}")
    update.message.reply_text("Request sent to admin.")

def approve(update: Update, context: CallbackContext):
    if update.message.from_user.id != ADMIN_ID: return
    user_id = int(context.args[0])
    requests_collection.update_one({"user_id": user_id}, {"$set": {"status": "approved"}})
    context.bot.send_message(chat_id=user_id, text="Request approved.")
    update.message.reply_text(f"User {user_id} approved.")

def reject(update: Update, context: CallbackContext):
    if update.message.from_user.id != ADMIN_ID: return
    user_id = int(context.args[0])
    requests_collection.update_one({"user_id": user_id}, {"$set": {"status": "rejected"}})
    context.bot.send_message(chat_id=user_id, text="Request rejected.")
    update.message.reply_text(f"User {user_id} rejected.")

def find_movie(update: Update, context: CallbackContext):
    user_id = update.message.from_user.id
    user_status = requests_collection.find_one({"user_id": user_id})
    if not user_status or user_status["status"] != "approved":
        update.message.reply_text("Request access using /connections.")
        return
    title = " ".join(context.args).lower()
    results = movies_collection.find({"title": {"$regex": title, "$options": "i"}})
    found = False
    for movie in results:
        update.message.reply_text(f"Title: {movie['title']}\nDescription: {movie['description']}")
        found = True
    if not found:
        update.message.reply_text("No movies found.")

def main():
    updater = Updater("YOUR_BOT_TOKEN")
    dispatcher = updater.dispatcher
    dispatcher.add_handler(CommandHandler("start", start))
    dispatcher.add_handler(CommandHandler("connections", connections))
    dispatcher.add_handler(CommandHandler("approve", approve))
    dispatcher.add_handler(CommandHandler("reject", reject))
    dispatcher.add_handler(CommandHandler("findmovie", find_movie))
    updater.start_polling()
    updater.idle()

if __name__ == "__main__":
    main()
    
