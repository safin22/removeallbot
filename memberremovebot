from pyrogram import Client, filters
from pyrogram.enums import ChatMemberStatus
import asyncio
from pyrogram.types import Message
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton, CallbackQuery
import json

# Set your bot token and main admin user ID
BOT_TOKEN = ""
API_ID =   # Load API ID from environment variable
API_HASH = ""  # Load API Hash from environment variable
MAIN_ADMIN_ID =   # Replace with main admin who can add sub admins your Telegram user ID
SUBADMIN_FILE = "subadmins.json"
running_removals = {}

# Initialize the bot
app = Client("RemoveBot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

def load_subadmins():
    """Loads the list of sub-admins from a JSON file."""
    try:
        with open(SUBADMIN_FILE, "r") as f:
            data = json.load(f)
            if isinstance(data, list):  # Ensure it's a list
                return data
            else:
                return []  # Return empty list if data is invalid
    except (FileNotFoundError, json.JSONDecodeError):
        return []  # Return empty list if file is missing or corrupted

# Save subadmins to the file
def save_subadmins(subadmins):
    with open(SUBADMIN_FILE, "w") as f:
        json.dump(subadmins, f)


async def is_admin(client, chat_id, user_id):
    """Check if a user is an admin."""
    try:
        member = await client.get_chat_member(chat_id, user_id)
        return member.status in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]
    except Exception:
        return False


async def bot_is_admin(client, chat_id):
    """Check if the bot is an admin with restrict permissions."""
    try:
        bot_member = await client.get_chat_member(chat_id, "me")
        return (bot_member.status in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]
                and bot_member.privileges.can_restrict_members)
    except Exception:
        return False


@app.on_message(filters.command("addadmin") & filters.group)
async def add_subadmin(client, message):
    """Adds a subadmin to the bot."""
    chat_id = message.chat.id
    sender_id = message.from_user.id

    if sender_id != MAIN_ADMIN_ID:
        await message.reply("❌ Only the main admin can add subadmins.")
        return

    await message.reply("⚠️ Please send the Telegram User ID of the subadmin.")

    @app.on_message(filters.text & filters.user(sender_id))
    async def confirm_subadmin(client, subadmin_message):
        try:
            subadmin_id = int(subadmin_message.text)
        except ValueError:
            await subadmin_message.reply("❌ Invalid User ID. Please send a valid Telegram User ID.")
            return

        subadmins = load_subadmins()
        if subadmin_id in subadmins:
            await subadmin_message.reply("❌ This user is already a subadmin.")
        else:
            subadmins.append(subadmin_id)
            save_subadmins(subadmins)
            await subadmin_message.reply(f"✅ {subadmin_id} has been added as a subadmin.")


@app.on_message(filters.command("removeadmin") & filters.group)
async def remove_subadmin(client, message):
    """Removes a subadmin from the bot."""
    chat_id = message.chat.id
    sender_id = message.from_user.id

    if sender_id != MAIN_ADMIN_ID:
        await message.reply("❌ Only the main admin can remove subadmins.")
        return

    await message.reply("⚠️ Please send the Telegram User ID of the subadmin to remove.")

    @app.on_message(filters.text & filters.user(sender_id))
    async def confirm_remove_subadmin(client, remove_message):
        try:
            subadmin_id = int(remove_message.text)
        except ValueError:
            await remove_message.reply("❌ Invalid User ID. Please send a valid Telegram User ID.")
            return

        subadmins = load_subadmins()
        if subadmin_id in subadmins:
            subadmins.remove(subadmin_id)
            save_subadmins(subadmins)
            await remove_message.reply(f"✅ {subadmin_id} has been removed as a subadmin.")
        else:
            await remove_message.reply(f"❌ {subadmin_id} is not a subadmin.")

@app.on_message(filters.command("listadmins") & filters.group)
async def list_subadmins(client, message):
    """Lists all subadmins."""
    subadmins = load_subadmins()
    if not subadmins:
        await message.reply("ℹ️ No subadmins have been added yet.")
    else:
        admin_list = "\n".join([f"- `{admin_id}`" for admin_id in subadmins])
        await message.reply(f"✅ **Subadmins:**\n{admin_list}")


@app.on_message(filters.command("removeall") & filters.group)
async def remove_all_members(client, message):
    """Request confirmation via inline buttons before removing members."""
    chat_id = message.chat.id
    sender_id = message.from_user.id

    # Check if user is an admin or subadmin
    if sender_id != MAIN_ADMIN_ID and sender_id not in load_subadmins():
        await message.reply("❌ You must be a subadmin or the main admin to use this command!")
        return

    if not await is_admin(client, chat_id, sender_id):
        await message.reply("❌ You must be an admin to use this command!")
        return

    if not await bot_is_admin(client, chat_id):
        await message.reply("❌ I need admin privileges with permission to restrict members!")
        return

    # Send confirmation message with inline buttons
    keyboard = InlineKeyboardMarkup([
        [InlineKeyboardButton("✅ YES", callback_data=f"confirm_remove|{chat_id}")],
        [InlineKeyboardButton("❌ Cancel", callback_data=f"cancel_remove|{chat_id}")]
    ])

    await message.reply("⚠️ Are you sure you want to remove all non-admin members?", reply_markup=keyboard)


@app.on_callback_query(filters.regex(r"confirm_remove\|(\-?\d+)"))
async def confirm_removal(client, callback_query: CallbackQuery):
    """Handles confirmation for member removal."""
    chat_id = int(callback_query.data.split("|")[-1])

    if chat_id in running_removals:
        await callback_query.answer("⚠️ Removal process is already running!", show_alert=True)
        return

    running_removals[chat_id] = True  # Mark removal as active

    await callback_query.message.edit_text("✅ Removing members... ⏳ Please wait...")

    removed_count = await perform_removal(client, chat_id)
    await callback_query.message.edit_text(f"✅ {removed_count} members removed successfully.")

    running_removals.pop(chat_id, None)  # Remove tracking once done


@app.on_callback_query(filters.regex(r"cancel_remove\|(\-?\d+)"))
async def cancel_removal(client, callback_query: CallbackQuery):
    """Cancels the removal process."""
    await callback_query.message.edit_text("❌ Removal process cancelled.")


async def perform_removal(client, chat_id):
    """Removes all non-admin members in batches while avoiding rate limits."""
    removed_count = 0
    members_to_remove = []

    async for member in client.get_chat_members(chat_id):
        if member.status not in [ChatMemberStatus.ADMINISTRATOR, ChatMemberStatus.OWNER]:
            members_to_remove.append(member.user.id)

    batch_size = 50  # Adjust batch size to prevent rate limits
    for i in range(0, len(members_to_remove), batch_size):
        if not running_removals.get(chat_id, False):
            return removed_count  # Stop if removal is canceled

        batch = members_to_remove[i:i + batch_size]
        tasks = [asyncio.create_task(remove_member(client, chat_id, user_id)) for user_id in batch]

        removed_in_batch = sum(await asyncio.gather(*tasks))
        removed_count += removed_in_batch
        await asyncio.sleep(5)  # Adjust delay to prevent Telegram bans

    return removed_count


async def remove_member(client, chat_id, user_id):
    """Helper function to remove a member."""
    try:
        await client.ban_chat_member(chat_id, user_id)
        await asyncio.sleep(2)  # Prevent spam actions
        await client.unban_chat_member(chat_id, user_id)
        return 1
    except:
        return 0

def get_admins():
    """Fetches sub-admin user IDs from the JSON file."""
    try:
        with open(SUBADMIN_FILE, "r") as f:
            data = json.load(f)
            return data if isinstance(data, list) else []
    except (FileNotFoundError, json.JSONDecodeError):
        return []

@app.on_message(filters.command("send", prefixes="/") & filters.group)
async def broadcast_to_admins(client: Client, message: Message):
    """Allow only the main admin and listed sub-admins to broadcast messages."""
    sender_id = message.from_user.id

    # Ensure command has a message
    if len(message.command) < 2:
        return await message.reply("⚠️ **Usage:** `/send Your Message`")

    # Get admin list from JSON file
    sub_admins = get_admins()

    # Combine main admin and sub-admins
    authorized_admins = set(sub_admins + [MAIN_ADMIN_ID])

    # Check if the sender is an authorized admin
    if sender_id not in authorized_admins:
        return await message.reply("❌ You are not authorized to use this command.")

    # Extract message text
    broadcast_message = " ".join(message.command[1:])  # Remove "/send" and get the rest

    # Send to all admins (excluding sender)
    sent_count = 0
    for admin_id in authorized_admins:
        if admin_id != sender_id:  # Skip the sender
            try:
                await app.send_message(admin_id, f"📢 **Admin Broadcast:**\n\n{broadcast_message}")
                sent_count += 1
            except Exception as e:
                print(f"Failed to message admin {admin_id}: {e}")

    await message.reply(f"✅ Message sent to {sent_count} admin(s).")

@app.on_message(filters.command("start"))
async def start(client, message):
    """Handles the start message."""
    start_message = (
        "🚀 **Welcome to Remove All Bot!** 🤖\n\n"
        "🔥 **Effortlessly remove all users from your group in one click!**\n\n"
        "🔹 **Commands & Features:**\n"
        "✅ `/removeall` - Remove all users instantly.\n"
        "👑 **Super Admin Only:**\n"
        "🔹 `/addadmin` - Grant sub-admin permissions.\n"
        "🔹 `/removeadmin` - Revoke sub-admin access.\n\n"
        "⚡ **Stay in control, manage your group like a boss!**\n\n"
        "🚀 **Powered by the one & only:** [@online_boss_safin_3](https://t.me/online_boss_safin_3)"
    )

    keyboard = InlineKeyboardMarkup(
        [
            [InlineKeyboardButton("🔥 Contact The Owner 🔥", url="https://t.me/online_boss_safin_3")],
            [InlineKeyboardButton("💬 Join Our Channel", url="https://t.me/+ibqwJysX04MyMGRl")]
        ]
    )

    await message.reply_text(start_message, reply_markup=keyboard, disable_web_page_preview=True)

print("Bot is running...")
app.run()
