import requests
from telegram import Update
from telegram.constants import ParseMode
from telegram.ext import Application, MessageHandler, filters, ContextTypes
import json
from datetime import datetime

BOT_TOKEN = "8145823685:AAHFgB6yMcl9d7cMk-sL_55IAos8Uo9vXu0"
ALLOWED_GROUP_ID = -1002421763830
cookies = {
    'source': 'mb',
    '_gid': 'GA1.2.1236421304.1706295770',
    '_gat_gtag_UA_137597827_4': '1',
    'session_key': 'hnl4y8xtfe918iiz2go67z85nsrvwqdn',
    '_ga': 'GA1.2.1006342705.1706295770',
    'datadome': '3AmY3lp~TL1WEuDKCnlwro_WZ1C6J66V1Y0TJ4ITf1Hvo4833Fh4LF3gHrPCKFJDPUPoXh2dXQHJ_uw0ifD8jmCaDltzE5T3zzRDbXOKH9rPNrTFs29DykfP3cfo7QGy',
    '_ga_R04L19G92K': 'GS1.1.1706295769.1.1.1706295794.0.0.0',
}

headers = {
    'Accept-Language': 'en-GB,en-US;q=0.9,en;q=0.8',
    'Connection': 'keep-alive',
    'Origin': 'https://shop.garena.sg',
    'Referer': 'https://shop.garena.sg/app/100067/idlogin',
    'Sec-Fetch-Dest': 'empty',
    'Sec-Fetch-Mode': 'cors',
    'Sec-Fetch-Site': 'same-origin',
    'User-Agent': 'Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36',
    'accept': 'application/json',
    'content-type': 'application/json',
    'sec-ch-ua': '"Not_A Brand";v="8", "Chromium";v="120"',
    'sec-ch-ua-mobile': '?1',
    'sec-ch-ua-platform': '"Android"',
    'x-datadome-clientid': 'DLm2W1ajJwdv~F~a_1d_1PyWnW6ns7GY5ChVcZY3HJ9r6D29661473aQaL2~3Nfh~Vf3m7rie7ObIb1_3eRN7J0G6uFZhMq5pM2jA828fE1dS7rZ7H3MWGQ5vGraAQWd',
}

def get_data(uid):
    json_data = {
        'app_id': 100067,
        'login_id': uid,
        'app_server_id': 0,
    }
    
    try:
        response = requests.post(
            'https://shop.garena.sg/api/auth/player_id_login', 
            cookies=cookies, 
            headers=headers, 
            json=json_data
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error occurred: {e}")
        return None

def get_profile_info(uid, region):
    url = f"https://player-info-xi.vercel.app/profile_info?uid={uid}&region={region.lower()}"
    try:
        response = requests.get(url)
        
        if not response.text.strip():
            return None
        
        if response.status_code != 200:
            return None
        try:
            data = json.loads(response.text)
            return data
        except json.JSONDecodeError:
            return None

    except requests.exceptions.RequestException as e:
        print(f"Request error: {e}")
        return None
    
def escape_markdown(text):
    special_characters = ['_', '*', '[', ']', '(', ')', '~', '`', '>', '#', '+', '-', '=', '|', '{', '}', '.', '!']
    for char in special_characters:
        text = text.replace(char, f"\\{char}")
    return text

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_GROUP_ID:
        await update.message.reply_text("This bot can only be used in the specified group.")
        return

    message_text = update.message.text
    
    if message_text.startswith("Get"):
        parts = message_text.split()
        
        if len(parts) != 2:
            await update.message.reply_text("Please provide UID in the format: Get UID")
            return
        
        uid = parts[1]
        
        player_data = get_data(uid)
        if not player_data:
            await update.message.reply_text("Failed to fetch player data.")
            return

        nickname = player_data.get("nickname", "Unknown")
        region = player_data.get("region", "N/A").lower()

        message = await update.message.reply_text(
            f"Fetching details for UID {uid}, nickname {escape_markdown(nickname)} in region {region.upper()}...",
            parse_mode=ParseMode.MARKDOWN,
        )

        profile_data = get_profile_info(uid, region)
        if not profile_data:
            await message.edit_text("Failed to fetch profile information.")
            return

        account_info = profile_data.get('AccountInfo', {})
        social_info = profile_data.get('socialinfo', {})
        guild_info = profile_data.get('GuildInfo', {})
        pet_info = profile_data.get('petInfo', {})
        honor_info = profile_data.get('creditScoreInfo', {})

        created_at_timestamp = account_info.get('AccountCreateTime', None)
        last_login_timestamp = account_info.get('AccountLastLogin', None)
        title_id = account_info.get('Title', None)
        elite_pass = account_info.get('hasElitePass', None)
        if elite_pass is True:
            gg = "Premium"
        else:
            gg = "Basic"
            
        created_at = datetime.fromtimestamp(created_at_timestamp).strftime('%d %B %Y %H:%M:%S') if created_at_timestamp else 'N/A'
        last_login = datetime.fromtimestamp(last_login_timestamp).strftime('%d %B %Y %H:%M:%S') if last_login_timestamp else 'N/A'
        title = f"https://garena420-api.vercel.app/item_details?id={title_id}"
        garena = requests.get(title)
        garena_json = garena.json()
        garena_420 = garena_json.get("Name", "Unknown Title")  
        formatted_message = (
            f"*ACCOUNT INFO:*\n\n"
            f"*🧑‍💻 ACCOUNT BASIC INFO*\n"
            f"├ Name: {escape_markdown(account_info.get('AccountName', 'N/A'))}\n"
            f"├ UID: {uid}\n"
            f"├ Level: {account_info.get('AccountLevel', 'N/A')} (Exp: {account_info.get('AccountEXP', 'N/A')})\n"
            f"├ Region: {account_info.get('AccountRegion', 'N/A')}\n"
            f"├ Honor Score: {honor_info.get('creditScore', 'N/A')}\n"
            f"├ Likes: {account_info.get('AccountLikes', 'N/A')}\n"
            f"├ Signature: {escape_markdown(social_info.get('AccountSignature', 'N/A'))}\n"
            f"└ Title: {garena_420}\n\n"
            f"*🎮 ACCOUNT ACTIVITY*\n"
            f"├ Fire Pass: {gg}\n"
            f"├ Current BP Badges: {account_info.get('AccountBPBadges', 'N/A')}\n"
            f"├ BR Rank: {account_info.get('BrRankPoint', 'N/A')}\n"
            f"├ CS Points: {account_info.get('CsRankPoint', 'N/A')}\n"
            f"├ Created At: {created_at}\n"
            f"└ Last Login: {last_login}\n\n"
            f"*👕 ACCOUNT OVERVIEW*\n"
            f"├ Avatar ID: {account_info.get('AccountAvatarId', 'N/A')}\n"
            f"├ Banner ID: {account_info.get('AccountBannerId', 'N/A')}\n"
            f"├ Equipped Skills: {profile_data.get('AccountProfileInfo', {}).get('EquippedSkills', 'N/A')}\n"
            f"└ Equipped Gun ID: {account_info.get('EquippedWeapon', 'N/A')}\n\n"
            f"*🐾 PET DETAILS*\n"
            f"├ Equipped?: {pet_info.get('isSelected' , 'N/A' )}\n"
            f"├ Pet Name: {escape_markdown(pet_info.get('name', 'N/A'))}\n"
            f"├ Pet Type:  {pet_info.get('id' , 'N/A' )}\n"
            f"├ Pet Exp:  {pet_info.get('exp' , 'N/A' )}\n"
            f"└ Pet Level:  {pet_info.get('level' , 'N/A' )}\n\n"
            f"*🛡️ GUILD INFO*\n"
            f"├ Guild Name: {escape_markdown(guild_info.get('GuildName', 'N/A'))}\n"
            f"├ Guild ID: {guild_info.get('GuildID', 'N/A')}\n"
            f"├ Guild Owner UID: {guild_info.get('GuildOwner', 'N/A')}\n"
            f"├ Guild Level: {guild_info.get('GuildLevel', 'N/A')}\n"
            f"├ Guild Capacity: {guild_info.get('GuildCapacity', 'N/A')}\n"
            f"└ Guild Members: {guild_info.get('GuildMember', 'N/A')}\n\n"
            f"[📢 Don't forget to join our channel for more updates](https://t.me/IShowAkiru5)"
        )

        await message.edit_text(formatted_message, parse_mode=ParseMode.MARKDOWN)

    elif message_text.startswith("Region"):
        parts = message_text.split()
        
        if len(parts) != 2:
            await update.message.reply_text("Please provide UID in the format: Region UID")
            return
        
        uid = parts[1]
        
        player_data = get_data(uid)
        if not player_data:
            await update.message.reply_text("Failed to fetch player data.")
            return
        
        nickname = player_data.get("nickname", "Unknown")
        region = player_data.get("region", "N/A").lower()

        response_message = f"User {escape_markdown(nickname)} with UID {uid} is in region {region.upper()}."
        await update.message.reply_text(response_message, parse_mode=ParseMode.MARKDOWN)

def main():
    application = Application.builder().token(BOT_TOKEN).build()
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    application.run_polling()

if __name__ == '__main__':
    main()
