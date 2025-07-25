require('dotenv').config();
const TelegramBot = require('node-telegram-bot-api');
const express = require('express');
const fs = require('fs-extra');

const bot = new TelegramBot(process.env.BOT_TOKEN, { polling: true });
const app = express();
const PORT = process.env.PORT || 3000;

const CHANNELS = ['@freeclaimltc', '@Konnetearnchannel'];
const ADMIN_ID = 6136849497;
const DATA_FILE = 'db.json';
const CLICK_REWARD = 0.0005;
const REF_REWARD = 0.01;
const WITHDRAW_LIMIT = 0.10;

let db = fs.readJsonSync(DATA_FILE);

function saveDB() {
  fs.writeJsonSync(DATA_FILE, db);
}

function getUser(id) {
  if (!db.users[id]) {
    db.users[id] = {
      balance: 0,
      referrals: [],
      clicks: [],
      joined: false,
    };
    saveDB();
  }
  return db.users[id];
}

async function checkChannels(userId) {
  for (let channel of CHANNELS) {
    const status = await bot.getChatMember(channel, userId);
    if (!['member', 'administrator', 'creator'].includes(status.status)) {
      return false;
    }
  }
  return true;
}

function mainMenu() {
  return {
    reply_markup: {
      keyboard: [
        ['💰 Balance', '📤 Withdraw'],
        ['🖱 Click and Earn', '📝 Signup and Earn'],
        ['📞 Contact Admin'],
      ],
      resize_keyboard: true,
    },
  };
}

bot.onText(/\/start (.+)/, async (msg, match) => {
  const user = getUser(msg.from.id);
  const ref = match[1];
  const joined = await checkChannels(msg.from.id);

  if (!joined) {
    return bot.sendMessage(msg.chat.id, `❗ Please join both channels before using the bot:\n\n` +
      CHANNELS.map(c => `👉 ${c}`).join('\n'));
  }

  user.joined = true;
  saveDB();

  if (ref && ref !== msg.from.id.toString()) {
    const refUser = getUser(ref);
    if (!refUser.referrals.includes(msg.from.id)) {
      refUser.balance += REF_REWARD;
      refUser.referrals.push(msg.from.id);
      bot.sendMessage(ref, `🎉 You earned $${REF_REWARD.toFixed(2)} from a new referral!`);
    }
  }

  return bot.sendMessage(msg.chat.id, `✅ Welcome to Dollarase Bot!`, mainMenu());
});

bot.onText(/\/start/, async (msg) => {
  const user = getUser(msg.from.id);
  const joined = await checkChannels(msg.from.id);

  if (!joined) {
    return bot.sendMessage(msg.chat.id, `❗ Please join both channels before using the bot:\n\n` +
      CHANNELS.map(c => `👉 ${c}`).join('\n'));
  }

  user.joined = true;
  saveDB();

  return bot.sendMessage(msg.chat.id, `✅ Welcome to Dollarase Bot!`, mainMenu());
});

bot.on('message', async (msg) => {
  const userId = msg.from.id;
  const text = msg.text;
  const user = getUser(userId);

  if (text === '💰 Balance') {
    return bot.sendMessage(msg.chat.id, `💼 Your balance: $${user.balance.toFixed(4)}`);
  }

  if (text === '📞 Contact Admin') {
    return bot.sendMessage(msg.chat.id, `📩 Contact admin at @Konnetearnchannel`);
  }

  if (text === '📤 Withdraw') {
    if (user.balance < WITHDRAW_LIMIT) {
      return bot.sendMessage(msg.chat.id, `❌ Minimum withdrawal is $${WITHDRAW_LIMIT.toFixed(2)}. Your balance: $${user.balance.toFixed(2)}`);
    }

    bot.sendMessage(msg.chat.id, `💳 Choose a payment method:`, {
      reply_markup: {
        keyboard: [['Payeer', 'Bank Account'], ['Litecoin', 'Opay']],
        resize_keyboard: true,
        one_time_keyboard: true
      }
    });

    bot.once('message', (msg2) => {
      const method = msg2.text;
      bot.sendMessage(msg.chat.id, `✍️ Now send your ${method} address:`);

      bot.once('message', (msg3) => {
        const address = msg3.text;
        const amount = user.balance;
        user.balance = 0;
        saveDB();

        const forwardMsg = `💸 *Withdrawal Request*\n\n👤 User: [${msg.from.first_name}](tg://user?id=${userId})\n💳 Method: ${method}\n📬 Address: ${address}\n💰 Amount: $${amount.toFixed(2)}`;
        bot.sendMessage(msg.chat.id, `✅ Withdrawal request submitted! You will be paid shortly.`);
        bot.sendMessage(ADMIN_ID, forwardMsg, { parse_mode: 'Markdown' });
        bot.sendMessage('@Konnetearnchannel', forwardMsg, { parse_mode: 'Markdown' });
      });
    });
  }

  if (text === '🖱 Click and Earn') {
    const now = Date.now();
    const clicks = user.clicks.filter(t => now - t < 15 * 60 * 60 * 1000);
    if (clicks.length >= 20) {
      const timeLeft = 15 * 60 * 60 * 1000 - (now - clicks[0]);
      const hrs = Math.floor(timeLeft / (1000 * 60 * 60));
      const mins = Math.floor((timeLeft % (1000 * 60 * 60)) / (1000 * 60));
      return bot.sendMessage(msg.chat.id, `⏱ You reached the limit. Try again in ${hrs}h ${mins}m.`);
    }

    const link = 'https://otieu.com/4/9574941';
    bot.sendMessage(msg.chat.id, `🖱 Click the link and wait for 60 seconds:\n${link}`);

    setTimeout(() => {
      user.balance += CLICK_REWARD;
      user.clicks.push(now);
      saveDB();
      bot.sendMessage(msg.chat.id, `✅ $${CLICK_REWARD.toFixed(4)} added to your balance.`);
    }, 60000);
  }

  if (text === '📝 Signup and Earn') {
    bot.sendMessage(msg.chat.id, `📝 *Earn by Signing Up*\n\n1. Click: https://otieu.com/4/8023804\n2. Enter your name\n3. Spin 3 times\n4. Click "Get Offer"\n5. Enter your email & sign up\n\nAfter signup, upload screenshot/proof here.`, { parse_mode: 'Markdown' });
  }

  if (msg.photo) {
    bot.sendMessage(ADMIN_ID, `🖼 *Signup Proof from* [${msg.from.first_name}](tg://user?id=${userId})`, {
      parse_mode: 'Markdown',
      reply_markup: {
        inline_keyboard: [
          [
            { text: '✅ Approve', callback_data: `approve_${userId}` },
            { text: '❌ Reject', callback_data: `reject_${userId}` }
          ]
        ]
      }
    });
    bot.forwardMessage(ADMIN_ID, msg.chat.id, msg.message_id);
    bot.sendMessage(msg.chat.id, `📤 Submitted! Please wait for admin approval.`);
  }
});

bot.on('callback_query', (query) => {
  const data = query.data;
  const adminId = query.from.id;

  if (adminId !== ADMIN_ID) return;

  if (data.startsWith('approve_')) {
    const userId = data.split('_')[1];
    bot.sendMessage(adminId, `💰 Enter amount to credit:`);

    bot.once('message', (msg2) => {
      const amount = parseFloat(msg2.text);
      const user = getUser(userId);
      user.balance += amount;
      saveDB();
      bot.sendMessage(userId, `✅ Your signup proof has been approved. You earned $${amount.toFixed(2)}!`);
      bot.sendMessage(adminId, `✅ Approved. $${amount.toFixed(2)} added to user ${userId}.`);
    });
  }

  if (data.startsWith('reject_')) {
    const userId = data.split('_')[1];
    bot.sendMessage(userId, `❌ Your signup proof was rejected. Make sure to complete signup properly and try again.`);
    bot.sendMessage(adminId, `❌ Rejected signup proof from user ${userId}.`);
  }
});

app.get('/', (req, res) => res.send('DollaraseBot is running!'));
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
