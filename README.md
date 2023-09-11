import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, CallbackContext, ConversationHandler, MessageHandler, CallbackQueryHandler, Filters  # Use Filters from telegram

# Enable logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)

# Define your bot's API Token here
API_TOKEN = "6376780320:AAGZuHi1MrVsIq7piUk0AE0I_wLIlJIVgyk"

# Define states for the conversation
START, CHOOSING, CUSTOMER_ISSUE, ANP_ISSUE, GST_ISSUE, GST_ENTERED = range(6)

# Define a dictionary of predefined responses based on user choices
choices = {
    "Customer issue": {
        "Recharge issue": "You chose Customer issue -> Recharge issue.",
        "Internet issue": "You chose Customer issue -> Internet issue.",
    },
    "ANP issue": {
        "ANP Disburse issue": "You chose ANP issue -> ANP Disburse issue.",
        "ANP Online issue": "You chose ANP issue -> ANP Online issue.",
    },
}

# Define user data dictionary to store entered GST number
user_data = {}

# Define a function to start the conversation
def start(update: Update, context: CallbackContext):
    update.message.reply_text("Welcome to Raj Chat Bot")
    return CHOOSING

# Define a function to generate the inline keyboard with buttons
def generate_keyboard(options):
    keyboard = [[InlineKeyboardButton(option, callback_data=option)] for option in options]
    return InlineKeyboardMarkup(keyboard)

# Define a function to handle button clicks
def button(update: Update, context: CallbackContext):
    query = update.callback_query
    query.answer()
    choice = query.data
    context.user_data['choice'] = choice
    next_options = choices.get(choice)
    if next_options:
        reply_text = "You chose {}. Choose an option:".format(choice)
        keyboard = generate_keyboard(next_options.keys())
        query.edit_message_text(text=reply_text, reply_markup=keyboard)
        return next_options_state(next_options)
    elif choice == "GST issue":
        update.message.reply_text("Please enter your GST Number:")
        return GST_ENTERED
    else:
        query.edit_message_text(text="Invalid choice.")
        return CHOOSING

# Define a function to handle GST number input
def handle_gst(update: Update, context: CallbackContext):
    gst_number = update.message.text
    user_data['gst_number'] = gst_number
    update.message.reply_text("GST Number {} saved for reference.".format(gst_number))
    
    # Send a knowledge message
    send_knowledge_message(update, context)
    
    return ConversationHandler.END

# Define a function to send knowledge message
def send_knowledge_message(update: Update, context: CallbackContext):
    gst_number = context.user_data.get('gst_number')
    if gst_number:
        knowledge_message = "Thank you for providing your GST Number: *{}*. Here's some knowledge for you:\n\n" \
                            "Insert your knowledge message here.".format(gst_number)
        update.message.reply_text(knowledge_message, parse_mode='Markdown')
    else:
        update.message.reply_text("GST Number not found in user data. Please enter your GST Number first.")

# Define a function to return the corresponding state based on selected options
def next_options_state(options):
    if "Recharge issue" in options:
        return CUSTOMER_ISSUE
    elif "ANP Disburse issue" in options:
        return ANP_ISSUE
    elif "GST issue" in options:
        return GST_ISSUE
    else:
        return CHOOSING

# Create an Updater and dispatcher
updater = Updater(token=API_TOKEN, use_context=True)
dispatcher = updater.dispatcher

# Register handlers
conv_handler = ConversationHandler(
    entry_points=[CommandHandler("start", start)],
    states={
        CHOOSING: [CallbackQueryHandler(button)],
        CUSTOMER_ISSUE: [CallbackQueryHandler(button)],
        ANP_ISSUE: [CallbackQueryHandler(button)],
        GST_ISSUE: [CallbackQueryHandler(button)],
        GST_ENTERED: [MessageHandler(Filters.text & ~Filters.command, handle_gst)],
    },
    fallbacks=[],
)
dispatcher.add_handler(conv_handler)

# Start your bot
updater.start_polling()
updater.idle()
