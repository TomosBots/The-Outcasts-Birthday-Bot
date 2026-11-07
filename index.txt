// The Outcasts Birthday Bot — Greek Mythology Edition
// Discord.js v14 | Railway-ready
//
// Required environment variables:
// DISCORD_TOKEN       = Bot token
// CLIENT_ID           = Discord application ID
// GUILD_ID            = The Outcasts server ID
//
// Optional:
// DATA_DIR            = /data (recommended Railway Volume mount path)
// TIMEZONE            = Europe/London
// BIRTHDAY_CHANNEL_ID = Optional channel ID for birthday announcements.
//                       If omitted, run /birthday-panel in the channel you want to use.
//
// Install: npm install discord.js
// Start:   node index.js

const {
  Client,
  GatewayIntentBits,
  Partials,
  REST,
  Routes,
  SlashCommandBuilder,
  PermissionFlagsBits,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  ModalBuilder,
  TextInputBuilder,
  TextInputStyle,
  EmbedBuilder,
  MessageFlags,
} = require("discord.js");
const fs = require("fs");
const path = require("path");

const TOKEN = process.env.DISCORD_TOKEN;
const CLIENT_ID = process.env.CLIENT_ID;
const GUILD_ID = process.env.GUILD_ID;
const DATA_DIR = process.env.DATA_DIR || "/data";
const TIMEZONE = process.env.TIMEZONE || "Europe/London";
const CONFIG_FILE = path.join(DATA_DIR, "birthday-config.json");
const BIRTHDAYS_FILE = path.join(DATA_DIR, "birthdays.json");

if (!TOKEN || !CLIENT_ID || !GUILD_ID) {
  console.error("Missing DISCORD_TOKEN, CLIENT_ID or GUILD_ID environment variable.");
  process.exit(1);
}

fs.mkdirSync(DATA_DIR, { recursive: true });

function readJson(file, fallback) {
  try {
    if (!fs.existsSync(file)) return fallback;
    return JSON.parse(fs.readFileSync(file, "utf8"));
  } catch (error) {
    console.error(`Could not read ${file}:`, error);
    return fallback;
  }
}
function writeJson(file, value) {
  const temp = `${file}.tmp`;
  fs.writeFileSync(temp, JSON.stringify(value, null, 2));
  fs.renameSync(temp, file);
}

let birthdays = readJson(BIRTHDAYS_FILE, {});
let config = readJson(CONFIG_FILE, { channelId: process.env.BIRTHDAY_CHANNEL_ID || null, lastRunDate: null });

const client = new Client({
  intents: [GatewayIntentBits.Guilds],
  partials: [Partials.Channel],
});

const commands = [
  new SlashCommandBuilder()
    .setName("birthday-panel")
    .setDescription("Post the Olympus birthday registration panel in this channel.")
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild),
  new SlashCommandBuilder()
    .setName("birthdays")
    .setDescription("See upcoming birthdays in The Outcasts."),
  new SlashCommandBuilder()
    .setName("birthday-remove")
    .setDescription("Remove a member's birthday registration.")
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageGuild)
    .addUserOption(option =>
      option.setName("member").setDescription("Member whose birthday should be removed").setRequired(true)
    ),
];

async function registerCommands() {
  const rest = new REST({ version: "10" }).setToken(TOKEN);
  await rest.put(Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID), {
    body: commands.map(command => command.toJSON()),
  });
  console.log("Birthday slash commands registered.");
}

function parseBirthday(input) {
  const match = /^(\d{2})-(\d{2})-(\d{4})$/.exec(input.trim());
  if (!match) return null;
  const day = Number(match[1]);
  const month = Number(match[2]);
  const year = Number(match[3]);
  const date = new Date(Date.UTC(year, month - 1, day));
  if (
    date.getUTCFullYear() !== year ||
    date.getUTCMonth() !== month - 1 ||
    date.getUTCDate() !== day ||
    year < 1900 ||
    year > new Date().getFullYear()
  ) return null;
  return { day, month, year, iso: `${String(year).padStart(4, "0")}-${match[2]}-${match[1]}` };
}

function datePartsInZone(date = new Date()) {
  const parts = new Intl.DateTimeFormat("en-GB", {
    timeZone: TIMEZONE,
    day: "2-digit",
    month: "2-digit",
    year: "numeric",
  }).formatToParts(date);
  const get = type => parts.find(part => part.type === type)?.value;
  return { day: Number(get("day")), month: Number(get("month")), year: Number(get("year")) };
}

function olympusPanel() {
  const embed = new EmbedBuilder()
    .setColor(0xC9A646)
    .setTitle("🏛️ THE OUTCASTS | HALL OF CELEBRATIONS")
    .setDescription(
      "Even the gods deserve a celebration!\n\n" +
      "Register your birthday below and let Olympus celebrate your special day. " +
      "When your birthday arrives, the gods will send a message worthy of a legend. ⚡\n\n" +
      "**How to register**\n" +
      "1. Press **Register My Birthday**.\n" +
      "2. Enter your date in **DD-MM-YYYY** format.\n" +
      "3. Submit and your birthday will be safely recorded.\n\n" +
      "*Your birth year is used only to validate your date. It will not be shown in birthday announcements.*"
    )
    .setFooter({ text: "THE OUTCASTS • HALL OF CELEBRATIONS" });
  const row = new ActionRowBuilder().addComponents(
    new ButtonBuilder()
      .setCustomId("birthday_register")
      .setLabel("Register My Birthday")
      .setEmoji("🎂")
      .setStyle(ButtonStyle.Primary)
  );
  return { embeds: [embed], components: [row] };
}

function birthdayWish(userId) {
  return new EmbedBuilder()
    .setColor(0xC9A646)
    .setTitle("⚡ THE GODS OF OLYMPUS HAVE SPOKEN! ⚡")
    .setDescription(
      `🏛️ **A VERY HAPPY BIRTHDAY, <@${userId}>!** 🎂\n\n` +
      "Today, Olympus shines a little brighter in your honour!\n\n" +
      "May the gods bless you with happiness, good fortune, strength and victories in the year ahead. " +
      "May your day be filled with laughter, unforgettable memories and all the things that make you smile.\n\n" +
      "You are a valued part of our family, and today we celebrate **YOU**. Have an absolutely legendary birthday!\n\n" +
      "**With love from the entire Outcasts family.** 🏛️✨"
    )
    .setFooter({ text: "THE OUTCASTS • ONE FAMILY, MANY LEGENDS" })
    .setTimestamp();
}

function formatUpcoming() {
  const today = datePartsInZone();
  const entries = Object.entries(birthdays).map(([userId, record]) => {
    const [year, month, day] = record.date.split("-").map(Number);
    let next = new Date(Date.UTC(today.year, month - 1, day));
    const todayDate = new Date(Date.UTC(today.year, today.month - 1, today.day));
    if (next < todayDate) next = new Date(Date.UTC(today.year + 1, month - 1, day));
    // In non-leap years, celebrate 29 February on 28 February.
    if (month === 2 && day === 29 && next.getUTCMonth() !== 1) {
      next = new Date(Date.UTC(today.year, 1, 28));
      if (next < todayDate) next = new Date(Date.UTC(today.year + 1, 1, 28));
    }
    return { userId, next, month, day };
  }).sort((a, b) => a.next - b.next).slice(0, 15);

  if (!entries.length) return "No birthdays have been registered yet.";
  return entries.map(item => {
    const dateText = new Intl.DateTimeFormat("en-GB", {
      timeZone: "UTC", day: "numeric", month: "long"
    }).format(item.next);
    return `🎂 **${dateText}** — <@${item.userId}>`;
  }).join("\n");
}

async function sendTodaysBirthdays() {
  const today = datePartsInZone();
  const dateKey = `${today.year}-${String(today.month).padStart(2, "0")}-${String(today.day).padStart(2, "0")}`;
  if (config.lastRunDate === dateKey) return;

  const channelId = config.channelId || process.env.BIRTHDAY_CHANNEL_ID;
  if (!channelId) {
    console.warn("No birthday channel configured. Run /birthday-panel in the desired channel.");
    return;
  }
  const channel = await client.channels.fetch(channelId).catch(() => null);
  if (!channel || !channel.isTextBased()) {
    console.error("Configured birthday channel was not found or is not a text channel.");
    return;
  }

  const todaysMembers = Object.entries(birthdays).filter(([, record]) => {
    const [year, month, day] = record.date.split("-").map(Number);
    const isLeapYear = new Date(Date.UTC(today.year, 1, 29)).getUTCMonth() === 1;
    const matches = month === today.month && day === today.day;
    const leapFallback = month === 2 && day === 29 && !isLeapYear && today.month === 2 && today.day === 28;
    return matches || leapFallback;
  });

  for (const [userId] of todaysMembers) {
    await channel.send({
      content: `🎉 <@${userId}>`,
      embeds: [birthdayWish(userId)],
      allowedMentions: { users: [userId] },
    }).catch(error => console.error(`Could not send birthday message for ${userId}:`, error));
  }

  config.lastRunDate = dateKey;
  writeJson(CONFIG_FILE, config);
  if (todaysMembers.length) console.log(`Sent ${todaysMembers.length} birthday greeting(s) for ${dateKey}.`);
}

client.once("ready", async () => {
  console.log(`🏛️ Birthday Bot online as ${client.user.tag}`);
  try {
    await registerCommands();
  } catch (error) {
    console.error("Could not register slash commands:", error);
  }
  // Check immediately on startup, then every 60 seconds. A date guard prevents duplicates.
  await sendTodaysBirthdays();
  setInterval(() => sendTodaysBirthdays().catch(console.error), 60_000);
});

client.on("interactionCreate", async interaction => {
  try {
    if (interaction.isChatInputCommand()) {
      if (interaction.commandName === "birthday-panel") {
        config.channelId = interaction.channelId;
        writeJson(CONFIG_FILE, config);
        await interaction.channel.send(olympusPanel());
        await interaction.reply({ content: "🏛️ The Hall of Celebrations panel has been posted and this channel is now set for birthday announcements.", flags: MessageFlags.Ephemeral });
        return;
      }

      if (interaction.commandName === "birthdays") {
        const embed = new EmbedBuilder()
          .setColor(0xC9A646)
          .setTitle("🏛️ UPCOMING BIRTHDAYS")
          .setDescription(formatUpcoming())
          .setFooter({ text: "THE OUTCASTS • HALL OF CELEBRATIONS" });
        await interaction.reply({ embeds: [embed], flags: MessageFlags.Ephemeral });
        return;
      }

      if (interaction.commandName === "birthday-remove") {
        const user = interaction.options.getUser("member", true);
        if (!birthdays[user.id]) {
          await interaction.reply({ content: "That member does not have a registered birthday.", flags: MessageFlags.Ephemeral });
          return;
        }
        delete birthdays[user.id];
        writeJson(BIRTHDAYS_FILE, birthdays);
        await interaction.reply({ content: `🏛️ Removed the birthday registration for ${user.tag}.`, flags: MessageFlags.Ephemeral });
        return;
      }
    }

    if (interaction.isButton() && interaction.customId === "birthday_register") {
      const modal = new ModalBuilder()
        .setCustomId("birthday_modal")
        .setTitle("🏛️ Register Your Birthday");
      const birthdayInput = new TextInputBuilder()
        .setCustomId("birthday_date")
        .setLabel("Your birthday (DD-MM-YYYY)")
        .setPlaceholder("21-11-2000")
        .setStyle(TextInputStyle.Short)
        .setMinLength(10)
        .setMaxLength(10)
        .setRequired(true);
      modal.addComponents(new ActionRowBuilder().addComponents(birthdayInput));
      await interaction.showModal(modal);
      return;
    }

    if (interaction.isModalSubmit() && interaction.customId === "birthday_modal") {
      const rawDate = interaction.fields.getTextInputValue("birthday_date");
      const parsed = parseBirthday(rawDate);
      if (!parsed) {
        await interaction.reply({
          content: "⚠️ That date doesn't look valid. Please use **DD-MM-YYYY**, for example **21-11-2000**.",
          flags: MessageFlags.Ephemeral,
        });
        return;
      }
      const previous = birthdays[interaction.user.id];
      birthdays[interaction.user.id] = {
        date: parsed.iso,
        registeredAt: new Date().toISOString(),
        updated: Boolean(previous),
      };
      writeJson(BIRTHDAYS_FILE, birthdays);
      await interaction.reply({
        content: previous
          ? "⚡ Your birthday has been updated in the Hall of Celebrations! Olympus will remember your special day."
          : "🏛️ Your birthday has been added to the Hall of Celebrations! We look forward to celebrating with you. 🎂",
        flags: MessageFlags.Ephemeral,
      });
    }
  } catch (error) {
    console.error("Interaction error:", error);
    const payload = { content: "Something went wrong. Please try again or contact the server team.", flags: MessageFlags.Ephemeral };
    if (interaction.deferred || interaction.replied) {
      await interaction.followUp(payload).catch(() => {});
    } else {
      await interaction.reply(payload).catch(() => {});
    }
  }
});

client.login(TOKEN);
