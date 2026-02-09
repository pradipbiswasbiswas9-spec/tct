
/**
 * SRK BOT – FULL VERSION
 * Features:
 * - Owner/Admin only commands
 * - Send photo with command
 * - Tag all members with different messages
 * - Reply, tag reply, group name change, text message
 */

const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require("@whiskeysockets/baileys");
const fs = require("fs");
const path = require("path");

const OWNER_NUMBER = "91XXXXXXXXXX"; // <-- CHANGE THIS
const BOT_NAME = "SRK BOT";

async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState("auth_info");

    const sock = makeWASocket({
        auth: state,
        printQRInTerminal: true,
        keepAliveIntervalMs: 15000
    });

    sock.ev.on("creds.update", saveCreds);

    sock.ev.on("messages.upsert", async ({ messages }) => {
        const msg = messages[0];
        if (!msg.message || msg.key.fromMe) return;

        const from = msg.key.remoteJid;
        const isGroup = from.endsWith("@g.us");
        const sender = msg.key.participant || from;

        const text =
            msg.message.conversation ||
            msg.message.extendedTextMessage?.text ||
            "";

        if (!text.startsWith("!")) return;

        const groupMeta = isGroup ? await sock.groupMetadata(from) : null;
        const admins = isGroup ? groupMeta.participants.filter(p => p.admin).map(p => p.id) : [];

        const isOwner = sender.includes(OWNER_NUMBER);
        const isAdmin = isGroup && admins.includes(sender);

        if (!isOwner && !isAdmin) return;

        const args = text.trim().split(" ");
        const cmd = args[0].toLowerCase();

        // HELLO
        if (cmd === "!hello") {
            await sock.sendMessage(from, { text: `👋 Hello! ${BOT_NAME} active hai 😎` });
        }

        // SEND PHOTO
        if (cmd === "!photo") {
            const photoPath = path.join(__dirname, "photo.jpg"); // add your photo here
            if (!fs.existsSync(photoPath)) {
                await sock.sendMessage(from, { text: "❌ photo.jpg file nahi mili" });
                return;
            }
            await sock.sendMessage(from, {
                image: fs.readFileSync(photoPath),
                caption: "SRK BOT PHOTO MESSAGE"
            });
        }

        // TEXT MESSAGE
        if (cmd === "!text") {
            const number = args[1];
            const message = args.slice(2).join(" ");
            await sock.sendMessage(number + "@s.whatsapp.net", { text: message });
        }

        // GROUP NAME CHANGE
        if (cmd === "!groupname" && isGroup) {
            const newName = args.slice(1).join(" ");
            await sock.groupUpdateSubject(from, newName);
        }

        // TAG ALL WITH DIFFERENT MESSAGES
        if (cmd === "!tagall" && isGroup) {
            const members = groupMeta.participants.map(p => p.id);

            let count = 1;
            for (let user of members) {
                await sock.sendMessage(from, {
                    text: `👋 Hey ${count}! Message from ${BOT_NAME}`,
                    mentions: [user]
                });
                count++;
            }
        }
    });

    console.log(`🤖 ${BOT_NAME} started...`);
}

startBot();
