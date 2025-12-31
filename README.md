# import { Client, Events, GatewayIntentBits, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle, AttachmentBuilder, SlashCommandBuilder, REST, Routes, ChatInputCommandInteraction } from "discord.js";
import { findBestMatch } from "string-similarity";
import { storage, type BotUserData, type BotCard } from "./storage";
import { createCanvas, loadImage, registerFont } from "canvas";

// Penalty directions
const PENALTY_DIRECTIONS = ["top-left", "top-center", "top-right", "bottom-left", "bottom-center", "bottom-right"];

// Generate penalty goal image
async function generatePenaltyImage(
  shootDirection: string | null,
  keeperDive: string | null,
  scored: boolean | null,
  shooterName: string,
  keeperName: string
): Promise<Buffer> {
  const WIDTH = 600;
  const HEIGHT = 400;
  const canvas = createCanvas(WIDTH, HEIGHT);
  const ctx = canvas.getContext("2d");

  // Green background (pitch)
  ctx.fillStyle = "#2d5a3f";
  ctx.fillRect(0, 0, WIDTH, HEIGHT);

  // Goal posts
  const goalX = 75;
  const goalWidth = WIDTH - 150;
  const goalY = 50;
  const goalHeight = 200;

  // Goal net (white lines)
  ctx.strokeStyle = "#ffffff";
  ctx.lineWidth = 8;
  ctx.strokeRect(goalX, goalY, goalWidth, goalHeight);

  // Crossbar
  ctx.fillStyle = "#ffffff";
  ctx.fillRect(goalX - 5, goalY - 5, goalWidth + 10, 10);
  
  // Posts
  ctx.fillRect(goalX - 5, goalY, 10, goalHeight + 5);
  ctx.fillRect(goalX + goalWidth - 5, goalY, 10, goalHeight + 5);

  // Net lines
  ctx.strokeStyle = "rgba(255, 255, 255, 0.3)";
  ctx.lineWidth = 1;
  for (let i = 0; i < 10; i++) {
    ctx.beginPath();
    ctx.moveTo(goalX + (goalWidth / 10) * i, goalY);
    ctx.lineTo(goalX + (goalWidth / 10) * i, goalY + goalHeight);
    ctx.stroke();
  }
  for (let i = 0; i < 5; i++) {
    ctx.beginPath();
    ctx.moveTo(goalX, goalY + (goalHeight / 5) * i);
    ctx.lineTo(goalX + goalWidth, goalY + (goalHeight / 5) * i);
    ctx.stroke();
  }

  // Draw 6 target zones
  const zones: Record<string, { x: number; y: number }> = {
    "top-left": { x: goalX + goalWidth * 0.17, y: goalY + goalHeight * 0.25 },
    "top-center": { x: goalX + goalWidth * 0.5, y: goalY + goalHeight * 0.25 },
    "top-right": { x: goalX + goalWidth * 0.83, y: goalY + goalHeight * 0.25 },
    "bottom-left": { x: goalX + goalWidth * 0.17, y: goalY + goalHeight * 0.75 },
    "bottom-center": { x: goalX + goalWidth * 0.5, y: goalY + goalHeight * 0.75 },
    "bottom-right": { x: goalX + goalWidth * 0.83, y: goalY + goalHeight * 0.75 },
  };

  // If no shot yet, show target zones
  if (!shootDirection) {
    for (const [dir, pos] of Object.entries(zones)) {
      ctx.beginPath();
      ctx.arc(pos.x, pos.y, 30, 0, Math.PI * 2);
      ctx.fillStyle = "rgba(255, 255, 0, 0.3)";
      ctx.fill();
      ctx.strokeStyle = "#ffff00";
      ctx.lineWidth = 2;
      ctx.stroke();
    }
  } else {
    // Show where shot went
    const shotPos = zones[shootDirection];
    if (shotPos) {
      // Ball
      ctx.beginPath();
      ctx.arc(shotPos.x, shotPos.y, 20, 0, Math.PI * 2);
      ctx.fillStyle = scored ? "#00ff00" : "#ff0000";
      ctx.fill();
      ctx.strokeStyle = "#ffffff";
      ctx.lineWidth = 3;
      ctx.stroke();

      // Keeper dive indicator
      if (keeperDive) {
        const keeperPos = zones[keeperDive];
        if (keeperPos) {
          ctx.beginPath();
          ctx.arc(keeperPos.x, keeperPos.y, 25, 0, Math.PI * 2);
          ctx.fillStyle = "rgba(255, 165, 0, 0.5)";
          ctx.fill();
          ctx.strokeStyle = "#ffa500";
          ctx.lineWidth = 3;
          ctx.stroke();
        }
      }
    }
  }

  // Header text
  ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
  ctx.fillRect(0, HEIGHT - 80, WIDTH, 80);
  
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 24px Arial";
  ctx.textAlign = "center";
  
  if (scored === null) {
    ctx.fillText("PENALTY!", WIDTH / 2, HEIGHT - 50);
    ctx.font = "16px Arial";
    ctx.fillText(`${shooterName} vs ${keeperName}`, WIDTH / 2, HEIGHT - 25);
  } else if (scored) {
    ctx.fillStyle = "#00ff00";
    ctx.fillText("GOAL!", WIDTH / 2, HEIGHT - 50);
    ctx.font = "16px Arial";
    ctx.fillStyle = "#ffffff";
    ctx.fillText(`${shooterName} scores!`, WIDTH / 2, HEIGHT - 25);
  } else {
    ctx.fillStyle = "#ff0000";
    ctx.fillText("SAVED!", WIDTH / 2, HEIGHT - 50);
    ctx.font = "16px Arial";
    ctx.fillStyle = "#ffffff";
    ctx.fillText(`${keeperName} makes the save!`, WIDTH / 2, HEIGHT - 25);
  }

  return canvas.toBuffer("image/png");
}

// Formation positions for 4-3-3 (x, y percentages of canvas)
const FORMATION_433: Record<string, { x: number; y: number }> = {
  "1": { x: 0.5, y: 0.88 },   // GK
  "2": { x: 0.15, y: 0.72 },  // LB
  "3": { x: 0.35, y: 0.72 },  // CB
  "4": { x: 0.65, y: 0.72 },  // CB
  "5": { x: 0.85, y: 0.72 },  // RB
  "6": { x: 0.25, y: 0.52 },  // CM
  "7": { x: 0.5, y: 0.52 },   // CM
  "8": { x: 0.75, y: 0.52 },  // CM
  "9": { x: 0.15, y: 0.28 },  // LW
  "10": { x: 0.5, y: 0.28 },  // ST
  "11": { x: 0.85, y: 0.28 }, // RW
};

// Rarity colors for cards
const RARITY_COLORS: Record<string, string> = {
  "Mythic": "#ffd700",
  "Legendary": "#38acec",
  "Epic": "#ffa500",
  "Rare": "#00bfff",
  "Common": "#90ee90"
};

// Stadium background paths
const STADIUM_BG_PATH = "./attached_assets/IMG_3302_1766846907560.jpeg";
const CARD_BG_PATH = "./attached_assets/IMG_3303_1766847373857.webp";

// Generate a single card image with stadium background
async function generateCardImage(
  card: { name: string; rating: number; pos: string; rarity: string; img: string },
  stats?: { goals: number; assists?: number; yellowCards?: number; redCards?: number; matches?: number }
): Promise<Buffer> {
  const WIDTH = 400;
  const HEIGHT = 500;
  const canvas = createCanvas(WIDTH, HEIGHT);
  const ctx = canvas.getContext("2d");

  // Try to load stadium background
  try {
    const bgImage = await loadImage(CARD_BG_PATH);
    ctx.drawImage(bgImage, 0, 0, WIDTH, HEIGHT);
    // Add dark overlay
    ctx.fillStyle = "rgba(0, 0, 0, 0.3)";
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
  } catch (err) {
    // Fallback gradient
    const gradient = ctx.createLinearGradient(0, 0, 0, HEIGHT);
    gradient.addColorStop(0, "#1a472a");
    gradient.addColorStop(1, "#0d2818");
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
  }

  const rarityColor = RARITY_COLORS[card.rarity] || "#666666";
  
  // Card frame
  const cardX = 50;
  
  // Card shadow
  ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
  ctx.fillRect(cardX + 5, cardY + 5, cardW, cardH);
  
  // Card gradient background
  const cardGradient = ctx.createLinearGradient(cardX, cardY, cardX, cardY + cardH);
  cardGradient.addColorStop(0, rarityColor);
  cardGradient.addColorStop(0.5, "#1a1a1a");
  cardGradient.addColorStop(1, "#000000");
  ctx.fillStyle = cardGradient;
  ctx.fillRect(cardX, cardY, cardW, cardH);
  
  // Card border
  ctx.strokeStyle = rarityColor;
  ctx.lineWidth = 4;
  ctx.strokeRect(cardX, cardY, cardW, cardH);

  // Try to load player image
  const imgSize = 200;
  const imgX = cardX + (cardW - imgSize) / 2;
  const imgY = cardY + 20;
  
  try {
    if (card.img && card.img.startsWith("http")) {
      const img = await loadImage(card.img);
      ctx.drawImage(img, imgX, imgY, imgSize, imgSize);
    } else {
      throw new Error("No image");
    }
  } catch {
    // Placeholder
    ctx.fillStyle = "#333333";
    ctx.fillRect(imgX, imgY, imgSize, imgSize);
    ctx.fillStyle = "#555555";
    ctx.beginPath();
    ctx.arc(imgX + imgSize / 2, imgY + 60, 40, 0, Math.PI * 2);
    ctx.fill();
    ctx.beginPath();
    ctx.ellipse(imgX + imgSize / 2, imgY + 150, 60, 40, 0, 0, Math.PI * 2);
    ctx.fill();
  }

  // Rating badge
  ctx.fillStyle = "#000000";
  ctx.fillRect(cardX + 10, cardY + 10, 50, 50);
  ctx.strokeStyle = rarityColor;
  ctx.lineWidth = 2;
  ctx.strokeRect(cardX + 10, cardY + 10, 50, 50);
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 28px Arial";
  ctx.textAlign = "center";
  ctx.fillText(String(card.rating), cardX + 35, cardY + 45);

  // Position badge
  ctx.fillStyle = "#000000";
  ctx.fillRect(cardX + cardW - 60, cardY + 10, 50, 30);
  ctx.strokeStyle = rarityColor;
  ctx.strokeRect(cardX + cardW - 60, cardY + 10, 50, 30);
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 16px Arial";
  ctx.fillText(card.pos, cardX + cardW - 35, cardY + 32);

  // Player name
  ctx.fillStyle = "rgba(0, 0, 0, 0.8)";
  ctx.fillRect(cardX, imgY + imgSize + 10, cardW, 40);
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 24px Arial";
  ctx.textAlign = "center";
  ctx.fillText(card.name, cardX + cardW / 2, imgY + imgSize + 38);

  // Rarity label
  ctx.fillStyle = rarityColor;
  ctx.font = "bold 18px Arial";
  ctx.fillText(card.rarity.toUpperCase(), cardX + cardW / 2, imgY + imgSize + 70);

  // Stats section if provided
  if (stats) {
    ctx.fillStyle = "rgba(0, 0, 0, 0.8)";
    ctx.fillRect(cardX, imgY + imgSize + 80, cardW, 60);
    
    ctx.fillStyle = "#ffffff";
    ctx.font = "14px Arial";
    ctx.textAlign = "center";
    
    const statsText = [
      `Goals: ${stats.goals || 0}`,
      `Assists: ${stats.assists || 0}`,
      `Matches: ${stats.matches || 0}`
    ].join("  |  ");
    ctx.fillText(statsText, cardX + cardW / 2, imgY + imgSize + 105);
    
    const cardsText = `Yellow: ${stats.yellowCards || 0}  |  Red: ${stats.redCards || 0}`;
    ctx.fillText(cardsText, cardX + cardW / 2, imgY + imgSize + 125);
  }

  // Bottom bar
  ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
  ctx.fillRect(0, HEIGHT - 50, WIDTH, 50);
  ctx.fillStyle = rarityColor;
  ctx.font = "bold 16px Arial";
  ctx.textAlign = "center";
  ctx.fillText("FRENDZ FOOTBALL CARDS", WIDTH / 2, HEIGHT - 20);

  return canvas.toBuffer("image/png");
}

// Generate team image
async function generateTeamImage(
  username: string,
  team: Record<string, BotCard>,
  chemistry: number
): Promise<Buffer> {
  const WIDTH = 900;
  const HEIGHT = 700;
  const canvas = createCanvas(WIDTH, HEIGHT);
  const ctx = canvas.getContext("2d");

  // Try to load stadium background
  try {
    const bgImage = await loadImage(STADIUM_BG_PATH);
    ctx.drawImage(bgImage, 0, 0, WIDTH, HEIGHT);
    // Add dark overlay for better card visibility
    ctx.fillStyle = "rgba(0, 0, 0, 0.4)";
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
  } catch (err) {
    // Fallback to gradient if image fails
    const gradient = ctx.createLinearGradient(0, 0, 0, HEIGHT);
    gradient.addColorStop(0, "#1a472a");
    gradient.addColorStop(0.5, "#2d5a3f");
    gradient.addColorStop(1, "#1a472a");
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, WIDTH, HEIGHT);
  }

  // Header bar
  ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
  ctx.fillRect(0, 0, WIDTH, 50);
  
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 20px Arial";
  ctx.textAlign = "left";
  ctx.fillText(username, 20, 32);
  
  // Calculate total rating
  const totalRating = Object.values(team).reduce((sum, c) => sum + c.rating, 0);
  const avgRating = Object.keys(team).length > 0 ? Math.round(totalRating / Object.keys(team).length) : 0;
  
  ctx.textAlign = "center";
  ctx.fillText(`OVR Rating: ${totalRating}`, WIDTH / 2, 32);
  
  // Chemistry indicator
  const chemColor = chemistry >= 80 ? "#00ff00" : chemistry >= 50 ? "#ffff00" : "#ff0000";
  ctx.fillStyle = chemColor;
  ctx.beginPath();
  ctx.arc(WIDTH - 40, 25, 12, 0, Math.PI * 2);
  ctx.fill();
  
  ctx.fillStyle = "#ffffff";
  ctx.textAlign = "right";
  ctx.fillText(`Chemistry: ${chemistry}`, WIDTH - 60, 32);

  // Draw player cards
  const CARD_WIDTH = 70;
  const CARD_HEIGHT = 95;

  for (const [pos, coords] of Object.entries(FORMATION_433)) {
    const card = team[pos];
    const x = coords.x * WIDTH - CARD_WIDTH / 2;
    const y = coords.y * HEIGHT - CARD_HEIGHT / 2 + 25; // offset for header

    if (card) {
      // Card background with rarity color
      const rarityColor = RARITY_COLORS[card.rarity] || "#666666";
      
      // Card shadow
      ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
      ctx.fillRect(x + 3, y + 3, CARD_WIDTH, CARD_HEIGHT);
      
      // Card gradient
      const cardGradient = ctx.createLinearGradient(x, y, x, y + CARD_HEIGHT);
      cardGradient.addColorStop(0, rarityColor);
      cardGradient.addColorStop(1, "#1a1a1a");
      ctx.fillStyle = cardGradient;
      ctx.fillRect(x, y, CARD_WIDTH, CARD_HEIGHT);
      
      // Card border
      ctx.strokeStyle = rarityColor;
      ctx.lineWidth = 2;
      ctx.strokeRect(x, y, CARD_WIDTH, CARD_HEIGHT);

      // Try to load player image
      let imageLoaded = false;
      if (card.img && card.img.startsWith("http")) {
        try {
          const img = await loadImage(card.img);
          ctx.drawImage(img, x + 5, y + 5, CARD_WIDTH - 10, CARD_WIDTH - 10);
          imageLoaded = true;
        } catch (err) {
          console.log(`Failed to load image for ${card.name}: ${err}`);
        }
      }
      
      if (!imageLoaded) {
        // Placeholder with player silhouette
        ctx.fillStyle = "#333333";
        ctx.fillRect(x + 5, y + 5, CARD_WIDTH - 10, CARD_WIDTH - 10);
        // Draw silhouette icon
        ctx.fillStyle = "#555555";
        ctx.beginPath();
        ctx.arc(x + CARD_WIDTH / 2, y + 20, 12, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.ellipse(x + CARD_WIDTH / 2, y + 48, 18, 12, 0, 0, Math.PI * 2);
        ctx.fill();
      }

      // Rating badge
      ctx.fillStyle = "#000000";
      ctx.fillRect(x + 3, y + 3, 22, 22);
      ctx.fillStyle = "#ffffff";
      ctx.font = "bold 14px Arial";
      ctx.textAlign = "center";
      ctx.fillText(String(card.rating), x + 14, y + 19);

      // Player name
      ctx.fillStyle = "#ffffff";
      ctx.font = "bold 9px Arial";
      ctx.textAlign = "center";
      const displayName = card.name.length > 10 ? card.name.substring(0, 9) + "." : card.name;
      ctx.fillText(displayName, x + CARD_WIDTH / 2, y + CARD_HEIGHT - 5);
      
      // Position label below card
      ctx.fillStyle = "rgba(0, 0, 0, 0.7)";
      ctx.fillRect(x + CARD_WIDTH / 2 - 15, y + CARD_HEIGHT + 2, 30, 16);
      ctx.fillStyle = "#ffffff";
      ctx.font = "bold 10px Arial";
      ctx.fillText(card.pos, x + CARD_WIDTH / 2, y + CARD_HEIGHT + 14);
    } else {
      // Empty slot
      ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
      ctx.fillRect(x, y, CARD_WIDTH, CARD_HEIGHT);
      ctx.strokeStyle = "rgba(255, 255, 255, 0.3)";
      ctx.lineWidth = 2;
      ctx.setLineDash([5, 5]);
      ctx.strokeRect(x, y, CARD_WIDTH, CARD_HEIGHT);
      ctx.setLineDash([]);
      
      // Position number
      ctx.fillStyle = "rgba(255, 255, 255, 0.5)";
      ctx.font = "bold 24px Arial";
      ctx.textAlign = "center";
      ctx.fillText(pos, x + CARD_WIDTH / 2, y + CARD_HEIGHT / 2 + 8);
    }
  }

  return canvas.toBuffer("image/png");
}

// Slash commands definition
const slashCommands = [
  new SlashCommandBuilder().setName("claim").setDescription("Claim a random football card"),
  new SlashCommandBuilder().setName("coins").setDescription("Check your coin balance"),
  new SlashCommandBuilder().setName("club").setDescription("View your club/inventory"),
  new SlashCommandBuilder().setName("team").setDescription("View your team formation"),
  new SlashCommandBuilder().setName("inv").setDescription("View your inventory"),
  new SlashCommandBuilder().setName("all").setDescription("View all available cards"),
  new SlashCommandBuilder().setName("result").setDescription("View recent match results"),
  new SlashCommandBuilder()
    .setName("buy")
    .setDescription("Buy a card")
    .addStringOption(option => option.setName("player").setDescription("Player name to buy").setRequired(true)),
  new SlashCommandBuilder()
    .setName("sell")
    .setDescription("Sell a card from your inventory")
    .addStringOption(option => option.setName("player").setDescription("Player name to sell").setRequired(true)),
  new SlashCommandBuilder()
    .setName("show")
    .setDescription("Show a card's details and stats")
    .addStringOption(option => option.setName("player").setDescription("Player name to show").setRequired(true)),
  new SlashCommandBuilder()
    .setName("teamadd")
    .setDescription("Add a player to your team")
    .addStringOption(option => option.setName("player").setDescription("Player name").setRequired(true))
    .addIntegerOption(option => option.setName("position").setDescription("Position (1-11)").setRequired(true).setMinValue(1).setMaxValue(11)),
  new SlashCommandBuilder()
    .setName("teamremove")
    .setDescription("Remove a player from your team")
    .addIntegerOption(option => option.setName("position").setDescription("Position (1-11)").setRequired(true).setMinValue(1).setMaxValue(11)),
  new SlashCommandBuilder()
    .setName("swap")
    .setDescription("Swap two team positions")
    .addIntegerOption(option => option.setName("pos1").setDescription("First position").setRequired(true).setMinValue(1).setMaxValue(11))
    .addIntegerOption(option => option.setName("pos2").setDescription("Second position").setRequired(true).setMinValue(1).setMaxValue(11)),
  new SlashCommandBuilder()
    .setName("friendly")
    .setDescription("Challenge someone to a match")
    .addUserOption(option => option.setName("opponent").setDescription("User to challenge").setRequired(true)),
  new SlashCommandBuilder()
    .setName("pack")
    .setDescription("Open a pack to get cards"),
];

const CARD_DATA = [
  // MYTHIC (0.1%)
  { name: "Poor_sip", rating: 96, pos: "CM", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235309825982749/IMG_3159.png" },
  { name: "Kaydenomg5", rating: 97, pos: "CAM", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235310060736677/IMG_3158.png" },
  { name: "Ironfudge", rating: 97, pos: "GK", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454376969583071274/IMG_3192.png" },
  { name: "AlmightyTwig1", rating: 97, pos: "ST", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454378980374347889/IMG_3198.png" },
  { name: "Poor_Sip", rating: 96, pos: "CM", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454322173408710820/IMG_3182.png" },
  { name: "SirTPE", rating: 96, pos: "CM", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454376082605215976/IMG_3190.png" },
  { name: "Nabil", rating: 96, pos: "ST", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454377659227443320/IMG_3194.png" },
  { name: "Ancientdragonxy", rating: 96, pos: "DF", rarity: "Mythic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454380074018148445/IMG_3202.png" },

  // LEGENDARY (2%)
  { name: "Kaydenomg5", rating: 95, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235310060736677/IMG_3158.png" },
  { name: "Annarozycka17", rating: 95, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454313721106268311/IMG_3169.png" },
  { name: "SirTPE", rating: 95, pos: "CM", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454321612185931848/IMG_3180.png" },
  { name: "AlmightyTwig1", rating: 95, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454382110457462899/IMG_3204.png" },
  { name: "Kaydenomg5", rating: 94, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235384224415745/IMG_3127.png" },
  { name: "Legendskiiii", rating: 94, pos: "CM", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454313236881997844/IMG_3167.png" },
  { name: "Darkdemonfighter", rating: 94, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416684076236822/IMG_3120.png" },
  { name: "AJ", rating: 93, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235310312657028/IMG_3157.png" },
  { name: "Dylan", rating: 93, pos: "DF", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454323022818447370/IMG_3184.png" },
  { name: "Ancientdragonxy", rating: 94, pos: "DF", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454542852167172177/IMG_3313.png" },
  { name: "Darkdemonfighter", rating: 95, pos: "ST", rarity: "Legendary", price: 0, img: "https://cdn.discordapp.com/attachments/1454490922023780362/1454818454535143444/IMG_3249.png" },
  // EPIC (12.9%)
  { name: "Bzndz", rating: 92, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235382576058600/IMG_3128.png" },
  { name: "Poor_Sip", rating: 92, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235381615824967/IMG_3130.png" },
  { name: "Alienspea", rating: 92, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454320950689665044/IMG_3178.png" },
  { name: "AlmightyTwig1", rating: 92, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416507718074501/IMG_3122.png" },
  { name: "Bzndz", rating: 92, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454234883789426781/IMG_3161.png" },
  { name: "AlmightyTwig1", rating: 92, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134819768331927612/Almighty.png" },
  { name: "Darkdemonfighter", rating: 92, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454412773101535284/IMG_3254.png" },
  { name: "Poor_Sip", rating: 92, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416684076236822/IMG_3120.png" },
  { name: "Annarozycka17", rating: 91, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235383272308757/IMG_3129.png" },
  { name: "Metalalta", rating: 91, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235380424376465/IMG_3131.png" },
  { name: "Arjannio", rating: 92, pos: "CM", rarity: "Epic", price: 0,
img: "https://cdn.discordapp.com/attachments/1454490922023780362/1454818453474250905/IMG_3256.png" },
  { name: "Mecha", rating: 91, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454378142264459306/IMG_3196.png" },
  { name: "Annarozycka17", rating: 91, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235310807453849/IMG_3156.png" },
  { name: "Poor_Sip", rating: 91, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454391121538056223/IMG_3218.png" },
  { name: "Kaydenomg5", rating: 91, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134876904613220392/Kayden.png" },
  { name: "Annarozycka17", rating: 91, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454411961470156953/IMG_3252.png" },
  { name: "Kaydenomg5", rating: 91, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454404733052653722/IMG_3244.png" },
  { name: "Ancientdragonxy", rating: 90, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235381028491304/IMG_3132.png" },
  { name: "Hamza", rating: 90, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454319699872714844/IMG_3176.png" },
  { name: "Darkdemonfighter", rating: 90, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454391743716786319/IMG_3220.png" },
  { name: "Poor_Sip", rating: 90, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134834547842891776/Sip.png" },
  { name: "Kaydenomg5", rating: 90, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416684348739685/IMG_3119.png" },
  { name: "Ancientdragonxy", rating: 90, pos: "DF", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454411232328286341/IMG_3250.png" },
  { name: "Darkdemonfighter", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235379900219466/IMG_3133.png" },
  { name: "Emrekiller24", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235358073061436/IMG_3135.png" },
  { name: "Mikeldowski", rating: 89, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235357775134852/IMG_3134.png" },
  { name: "Jellyman14327", rating: 89, pos: "GK", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235357523611718/IMG_3136.png" },
  { name: "Pokemonolly10", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235357288726639/IMG_3138.png" },
  { name: "Nabil", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454226186505814106/IMG_3100.png" },
  { name: "AlmightyTwig1", rating: 89, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454226758936039534/IMG_3102.png" },
  { name: "Darkdemonfighter", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235310568374333/IMG_3155.png" },
  { name: "TheDoctor", rating: 89, pos: "CM", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454382786248048791/IMG_3210.png" },
  { name: "Jellyman14327", rating: 89, pos: "GK", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454393048040472608/IMG_3225.png" },
  { name: "Darkdemonfighter", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134819302021791824/Dark.png" },
  { name: "Nabil", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454410623579324557/IMG_3248.png" },
  { name: "Emrekiller24", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895285349400637/Emre.png" },
  { name: "AJ", rating: 89, pos: "ST", rarity: "Epic", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416685036605440/IMG_3115.png" },

  // RARE (35%)
  { name: "AstroDami1", rating: 86, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235355837497456/IMG_3144.png" },
  { name: "Furious118", rating: 86, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235356063993908/IMG_3142.png" },
  { name: "PZ", rating: 86, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235329065123962/IMG_3145.png" },
  { name: "Alienspea", rating: 86, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235328415273000/IMG_3146.png" },
  { name: "Legendskiiii", rating: 86, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134817397581287514/Legendskiii.png" },
  { name: "Darkdemonfighter", rating: 86, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454389655255384198/IMG_3214.png" },
  { name: "Mikeldowski", rating: 86, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454400374415949960/IMG_3229.png" },
  { name: "Mercy", rating: 86, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416683853942996/IMG_3121.png" },
  { name: "Ironfudge", rating: 86, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454415732690649182/IMG_3262.png" },
  { name: "Power", rating: 87, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235327882596353/IMG_3149.png" },
  { name: "Joshua", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235311163834529/IMG_3154.png" },
  { name: "Power", rating: 86, pos: "DF", rarity: "Rare", price: 0,img: "https://cdn.discordapp.com/attachments/1454490922023780362/1454818454015185032/IMG_3255.png" },
  { name: "AlmightyTwig1", rating: 87, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235356315517220/IMG_3140.png" },
  { name: "Legendskiiii", rating: 87, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235356550529088/IMG_3141.png" },
  { name: "Pz", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454226753206882467/IMG_3103.png" },
  { name: "Ironfudge", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454226395604586589/IMG_3101.png" },
  { name: "Mikeldowski", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454234783570989108/IMG_3165.png" },
  { name: "Dylan", rating: 87, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454382766128103656/IMG_3209.png" },
  { name: "Emrekiller24", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454393604461166644/IMG_3227.png" },
  { name: "Ancientdragonxy", rating: 87, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134875223540379788/Ancient.png" },
  { name: "Ironfudge", rating: 87, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454413824844365945/IMG_3256.png" },
  { name: "Ancientdragonxy", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416082759581828/IMG_3126.png" },
  { name: "Mercy", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895231389683783/Aouarrr.png" },
  { name: "Nabil", rating: 87, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895233512001637/Nabil.png" },
  { name: "X0X", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235312061415606/IMG_3153.png" },
  { name: "AJ", rating: 88, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235356789608611/IMG_3137.png" },
  { name: "Zeph", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235328150896761/IMG_3148.png" },
  { name: "Serpent", rating: 88, pos: "GK", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235357036937237/IMG_3139.png" },
  { name: "Zeph", rating: 88, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235309549293668/IMG_3160.png" },
  { name: "Mecha", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454382752148357276/IMG_3208.png" },
  { name: "Mercy", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454393359056502784/IMG_3226.png" },
  { name: "AJ", rating: 88, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895230806655017/Immenseaj.png" },
  { name: "Annarozycka17", rating: 88, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895287492685945/ANNA.png" },
  { name: "Jellyman14327", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895285013848176/Jelly.png" },
  { name: "Jellyman14327", rating: 88, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416684797657145/IMG_3116.png" },
  { name: "Iic", rating: 85, pos: "ST", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235327601573969/IMG_3150.png" },
  { name: "Kaydenomg5", rating: 85, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454390329993199736/IMG_3216.png" },
  { name: "Ironfudge", rating: 85, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134822906258542662/Iron.png" },
  { name: "CJ", rating: 85, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895231901368371/CJ.png" },
  { name: "Mikeldowski", rating: 85, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895287840809151/Mikel.png" },
  { name: "Pz", rating: 85, pos: "CM", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454415262433546426/IMG_3259.png" },
  { name: "Power", rating: 85, pos: "DF", rarity: "Rare", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416322413723708/IMG_3124.png" },

  // COMMON (50%)
  { name: "Jamie", rating: 80, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235312552280064/IMG_3151.png" },
  { name: "Arjannio", rating: 80, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235311562428499/IMG_3152.png" },
  { name: "Joshua", rating: 80, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895233973354627/Joshua.png" },
  { name: "Pingu", rating: 82, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235328805081230/IMG_3147.png" },
  { name: "Aguero", rating: 84, pos: "GK", rarity: "Common",price:   0,img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454542852708241591/IMG_3311.png" },
  { name: "Azan", rating: 82, pos: "DF", rarity: "Common" , price: 0,
    img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454542852444127484/IMG_3312.png" },
  { name: "Alienspea", rating: 82, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454388702909436037/IMG_3212.png" },
  { name: "Furious118", rating: 83, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134866995117047860/Furi.png" },
  { name: "Dylan", rating: 83, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895284623781888/Dylan.png" },
  { name: "Metal", rating: 84, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134815607292969030/Metal_3.png" },
  { name: "Joshua", rating: 84, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454400596382716089/IMG_3231.png" },
  { name: "Alienspea", rating: 84, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454401738382770319/IMG_3238.png" },
  { name: "Furious118", rating: 84, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454401213222359132/IMG_3236.png" },
  { name: "Ironfudge", rating: 84, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454403531476828331/IMG_3242.png?ex=69539957&is=695247d7&hm=4b01aa9293d3be5c8037371d580940cc38d6dfc0a424a6994be0038966f6fcd7&" },
  { name: "Metal", rating: 84, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416684566974559/IMG_3117.png" },
  { name: "Nabil", rating: 84, pos: "ST", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454416685309366433/IMG_3114.png" },
  { name: "Pingu", rating: 72, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454235328805081230/IMG_3147.png" },
  { name: "Pingu", rating: 74, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454405681116479488/IMG_3246.png" },
  { name: "Pingu", rating: 69, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895232895418558/PINGU.png" },
  { name: "Sun", rating: 81, pos: "GK", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895230303346851/SUN.png" },
  { name: "Alienspea", rating: 81, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1454220184008261736/1454401176979116144/IMG_3235.png" },
  { name: "Emman", rating: 79, pos: "DF", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134895232304041994/Rk2sak.png" },
  { name: "SirTPE", rating: 76, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134875222873477210/SIRTPE.png" },
  { name: "Hamza", rating: 75, pos: "CM", rarity: "Common", price: 0, img: "https://cdn.discordapp.com/attachments/1132638459249565739/1134897344224170074/Hamza.png" }
];

// In-memory cache backed by database
const userCache = new Map<string, BotUserData>();

async function getUser(id: string): Promise<BotUserData> {
  if (!userCache.has(id)) {
    const userData = await storage.loadBotUser(id);
    userCache.set(id, userData);
  }
  return userCache.get(id)!;
}

async function saveUser(id: string): Promise<void> {
  const userData = userCache.get(id);
  if (userData) {
    await storage.saveBotUser(id, userData);
  }
}

function rarityRoll(): string {
    const roll = Math.floor(Math.random() * 1000) + 1;

    if (roll === 1) {
        return "Mythic";
    } else if (roll <= 21) {       // 1 + 20
        return "Legendary";
    } else if (roll <= 150) {      // 21 + 129
        return "Epic";
    } else if (roll <= 500) {      // 150 + 350
        return "Rare";
    } else {
        return "Common";
    }
}

function getRarityColor(rarity: string): number {
  const colors: Record<string, number> = {
    Mythic: 0xFFD700,
    Legendary: 0x1520A6,
    Epic: 0xFF2400,
    Rare: 0xB0FC38,
    Common: 0x888888
  };
  return colors[rarity] || 0x2ecc71;
}

function priceFromRating(r: number): number {
  let p = Math.floor(35000 * Math.pow(1.081, r - 69));
  if (p < 35000) p = 35000;
  if (p > 3780000) p = 3780000;
  return p;
}

function getPackCost(rarity: string): number {
  switch (rarity) {
    case "Mythic": return 1250000;
    case "Legendary": return 750000;
    case "Epic": return 375000;
    case "Rare": return 100000;
    case "Common": return 20000;
    default: return 50000;
  }
}

function findCard(query: string) {
  const names = CARD_DATA.map(c => c.name);
  const match = findBestMatch(query, names).bestMatch.target;
  return CARD_DATA.find(c => c.name === match);
}

async function seedCards() {
  try {
    const existingCards = await storage.getRandomCard().catch(() => null);
    if (!existingCards) {
      console.log("Cards seeded successfully");
    }
  } catch (e) {
    console.error("Error seeding cards:", e);
  }
}

export async function setupBot() {
  if (!process.env.DISCORD_TOKEN) {
    console.log("No DISCORD_TOKEN found, skipping bot setup");
    return;
  }

  const client = new Client({
    intents: [
      GatewayIntentBits.Guilds,
      GatewayIntentBits.GuildMessages,
      GatewayIntentBits.MessageContent
    ]
  });

  client.once(Events.ClientReady, async c => {
    console.log(`Ready! Logged in as ${c.user.tag}`);
    storage.createLog({ level: "info", message: `Bot logged in as ${c.user.tag}` });
    
    // Register slash commands
    try {
      const rest = new REST({ version: "10" }).setToken(process.env.DISCORD_TOKEN!);
      console.log("Registering slash commands...");
      await rest.put(
        Routes.applicationCommands(c.user.id),
        { body: slashCommands.map(cmd => cmd.toJSON()) }
      );
      console.log("Slash commands registered successfully!");
    } catch (error) {
      console.error("Error registering slash commands:", error);
    }
  });

  // Handle mention-based commands
  client.on(Events.MessageCreate, async (message) => {
    if (message.author.bot) return;
    
    // Check if bot is mentioned
    const botMention = `<@${client.user?.id}>`;
    const botMentionAlt = `<@!${client.user?.id}>`;
    
    if (!message.content.startsWith(botMention) && !message.content.startsWith(botMentionAlt)) return;

    // Remove mention and get command
    const content = message.content.replace(botMention, "").replace(botMentionAlt, "").trim();
    const args = content.split(" ");
    const cmd = args.shift()?.toLowerCase();
    const userId = message.author.id;
    const user = await getUser(userId);

    try {
      // CLAIM
      if (cmd === "claim") {
        const now = Date.now();

        if (now < user.cooldown) {
          return await message.reply(`⏳ Claim again <t:${Math.floor(user.cooldown / 1000)}:R>`);
        }

        user.cooldown = now + 3600000;
        await saveUser(userId);

        const rarity = rarityRoll();
        const pool = CARD_DATA.filter(c => c.rarity === rarity);
        const card = pool[Math.floor(Math.random() * pool.length)];

        // Generate card image with stadium background
        const cardImage = await generateCardImage(card);
        const attachment = new AttachmentBuilder(cardImage, { name: "card.png" });

        const embed = new EmbedBuilder()
          .setColor(getRarityColor(rarity))
          .setTitle("Football Card Claimed")
          .setDescription(`**${card.name}**\n Rating: ${card.rating}\n Position: ${card.pos}\n Rarity: ${card.rarity}`)
          .setImage("attachment://card.png");

        const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
          new ButtonBuilder()
            .setCustomId(`keep_${userId}`)
            .setLabel("Keep")
            .setStyle(ButtonStyle.Success),
          new ButtonBuilder()
            .setCustomId(`sell_${userId}`)
            .setLabel("Sell")
            .setStyle(ButtonStyle.Danger)
        );

        const sent = await message.reply({ embeds: [embed], files: [attachment], components: [row] });
        const collector = sent.createMessageComponentCollector({ time: 60000 });

        collector.on("collect", async (i) => {
          if (i.user.id !== userId) return;

          if (i.customId === `keep_${userId}`) {
            user.inventory.push({ name: card.name, rating: card.rating, pos: card.pos, rarity: card.rarity, img: card.img });
            await saveUser(userId);
            i.update({ content: "Card added to inventory", embeds: [], components: [], files: [] });
            storage.createLog({ level: "info", message: `${message.author.tag} kept card: ${card.name}` });
          }

          if (i.customId === `sell_${userId}`) {
            const sellPrice = Math.floor(priceFromRating(card.rating) / 2);
            user.coins += sellPrice;
            await saveUser(userId);
            i.update({ content: `Sold for ${sellPrice.toLocaleString()} coins`, embeds: [], components: [], files: [] });
            storage.createLog({ level: "info", message: `${message.author.tag} sold card: ${card.name}` });
          }

          collector.stop();
        });

        storage.createLog({ level: "info", message: `${message.author.tag} claimed a card: ${card.name}` });
      }
      // BUY with carousel
      if (cmd === "buy") {
        const query = args.join(" ");
        if (!query) return await message.reply("❌ Use `!buy name`");

        let matches = CARD_DATA.filter(c => c.name.toLowerCase().includes(query.toLowerCase()));
        if (!matches.length) return await message.reply("❌ Card not found");

        matches.sort((a, b) => a.rating - b.rating);
        let index = 0;

        const showCard = () => {
          const c = matches[index];
          const price = priceFromRating(c.rating);
          return new EmbedBuilder()
            .setColor(getRarityColor(c.rarity))
            .setTitle(`Buy ${c.name}`)
            .setDescription(`⭐ ${c.rating}\n🎯 ${c.pos}\n💎 ${c.rarity}\n💰 ${price.toLocaleString()} coins\n\n[${index + 1}/${matches.length}]`)
            .setImage(c.img);
        };

        const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
          new ButtonBuilder().setCustomId("prev").setLabel("◀").setStyle(ButtonStyle.Secondary),
          new ButtonBuilder().setCustomId("buy").setLabel("Buy").setStyle(ButtonStyle.Success),
          new ButtonBuilder().setCustomId("next").setLabel("▶").setStyle(ButtonStyle.Secondary)
        );

        const sent = await message.reply({ embeds: [showCard()], components: [row] });
        const collector = sent.createMessageComponentCollector({ time: 60000 });

        collector.on("collect", async (i) => {
          if (i.user.id !== userId) return;

          if (i.customId === "prev") index = (index - 1 + matches.length) % matches.length;
          if (i.customId === "next") index = (index + 1) % matches.length;

          if (i.customId === "buy") {
            const c = matches[index];
            const price = priceFromRating(c.rating);
            if (user.coins < price) {
              return i.update({ content: "❌ Not enough coins", components: [] });
            }
            user.coins -= price;
            user.inventory.push({ name: c.name, rating: c.rating, pos: c.pos, rarity: c.rarity, img: c.img });
            await saveUser(userId);
            i.update({ content: "✅ Purchase successful", components: [] });
            storage.createLog({ level: "info", message: `${message.author.tag} bought card: ${c.name}` });
            collector.stop();
            return;
          }

          i.update({ embeds: [showCard()], components: [row] });
        });
      }

      // CLUB (renamed from inventory)
      if (cmd === "club") {
        if (user.inventory.length === 0) {
          return await message.reply("📦 Your club is empty.");
        }
        const inv = user.inventory.map(c => `${c.name} (${c.rating})`).join("\n");
        await message.reply(`📦 **Club**\n${inv}\n\n💰 **Balance**: ${user.coins.toLocaleString()} coins`);
        storage.createLog({ level: "info", message: `${message.author.tag} viewed club` });
      }

      // COINS
      if (cmd === "coins") {
        await message.reply(`💰 You have ${user.coins.toLocaleString()} coins`);
      }

      // SHOW
      if (cmd === "show") {
        const name = args.join(" ");
        const names = user.inventory.map(c => c.name);
        const match = findBestMatch(name, names).bestMatch?.target;
        const card = user.inventory.find(c => c.name === match);
        if (!card) return await message.reply("You don't own this card.");

        const stats = user.stats[card.name] || { goals: 0, assists: 0, yellowCards: 0, redCards: 0, matches: 0 };
        const price = priceFromRating(card.rating);
        
        // Generate card image with stats
        const cardImage = await generateCardImage(card, stats);
        const attachment = new AttachmentBuilder(cardImage, { name: "card.png" });

        const embed = new EmbedBuilder()
          .setColor(getRarityColor(card.rarity))
          .setTitle(`${card.name}`)
          .setDescription(`**Rating**: ${card.rating}\n**Position**: ${card.pos}\n**Rarity**: ${card.rarity}\n**Worth**: ${price.toLocaleString()} coins`)
          .setImage("attachment://card.png");

        await message.reply({ embeds: [embed], files: [attachment] });
      }

      // SELL with carousel for duplicates
      if (cmd === "sell") {
        const name = args.join(" ");
        if (!name) return await message.reply("❌ Use `!sell <player>`");

        const names = user.inventory.map(c => c.name);
        const match = findBestMatch(name, names).bestMatch?.target;
        
        let matches = user.inventory
          .map((card, idx) => ({ card, idx }))
          .filter(item => item.card.name === match);
        
        if (!matches.length) return await message.reply("❌ You don't own this player.");

        let index = 0;

        const showCard = () => {
          const { card } = matches[index];
          const sellPrice = Math.floor(priceFromRating(card.rating) / 2);
          return new EmbedBuilder()
            .setColor(getRarityColor(card.rarity))
            .setTitle(`Sell ${card.name}`)
            .setDescription(`⭐ ${card.rating}\n🎯 ${card.pos}\n💎 ${card.rarity}\n💰 ${sellPrice.toLocaleString()} coins\n\n[${index + 1}/${matches.length}]`)
            .setImage(card.img);
        };

        const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
          new ButtonBuilder().setCustomId(`sell_prev_${userId}`).setLabel("◀").setStyle(ButtonStyle.Secondary),
          new ButtonBuilder().setCustomId(`sell_confirm_${userId}`).setLabel("Sell").setStyle(ButtonStyle.Danger),
          new ButtonBuilder().setCustomId(`sell_next_${userId}`).setLabel("▶").setStyle(ButtonStyle.Secondary)
        );

        const sent = await message.reply({ embeds: [showCard()], components: [row] });
        const collector = sent.createMessageComponentCollector({ time: 60000 });

        collector.on("collect", async (i) => {
          if (i.user.id !== userId) return;

          if (i.customId === `sell_prev_${userId}`) index = (index - 1 + matches.length) % matches.length;
          if (i.customId === `sell_next_${userId}`) index = (index + 1) % matches.length;

          if (i.customId === `sell_confirm_${userId}`) {
            const { card, idx } = matches[index];
            const sellPrice = Math.floor(priceFromRating(card.rating) / 2);

            // Remove from team if they're on it
            for (const [pos, teamCard] of Object.entries(user.team)) {
              if (teamCard.name === card.name && teamCard.rating === card.rating) {
                delete user.team[pos];
                break;
              }
            }

            // Remove from inventory and add coins
            user.inventory.splice(idx, 1);
            user.coins += sellPrice;
            await saveUser(userId);

            i.update({ content: `💰 Sold **${card.name}** for ${sellPrice.toLocaleString()} coins`, embeds: [], components: [] });
            storage.createLog({ level: "info", message: `${message.author.tag} sold card: ${card.name} for ${sellPrice} coins` });
            collector.stop();
            return;
          }

          i.update({ embeds: [showCard()], components: [row] });
        });
      }

      // PACK
      if (cmd === "pack") {
        let selectedRarity = "";
        let selectedCard: typeof CARD_DATA[0] | null = null;
        let packCost = 0;

        const showRarityButtons = () => {
          return new ActionRowBuilder<ButtonBuilder>().addComponents(
            new ButtonBuilder().setCustomId(`pack_mythic_${userId}`).setLabel("Mythic (500k)").setStyle(ButtonStyle.Danger),
            new ButtonBuilder().setCustomId(`pack_legendary_${userId}`).setLabel("Legendary (250k)").setStyle(ButtonStyle.Primary),
            new ButtonBuilder().setCustomId(`pack_epic_${userId}`).setLabel("Epic (125k)").setStyle(ButtonStyle.Secondary),
            new ButtonBuilder().setCustomId(`pack_rare_${userId}`).setLabel("Rare (75k)").setStyle(ButtonStyle.Secondary),
            new ButtonBuilder().setCustomId(`pack_common_${userId}`).setLabel("Common (35k)").setStyle(ButtonStyle.Secondary)
          );
        };

        const showOpenButton = () => {
          return new ActionRowBuilder<ButtonBuilder>().addComponents(
            new ButtonBuilder().setCustomId(`pack_open_${userId}`).setLabel("Open Pack").setStyle(ButtonStyle.Success),
            new ButtonBuilder().setCustomId(`pack_back_${userId}`).setLabel("Back").setStyle(ButtonStyle.Secondary)
          );
        };

        const embed = new EmbedBuilder()
          .setColor(0xFFD700)
          .setTitle("🏆 FUT Champions Pack")
          .setDescription("Choose which rarity pack you want to open!")
          .setImage("https://cdn.discordapp.com/attachments/1454220184008261736/1454235309825982749/IMG_3159.png");

        const sent = await message.reply({ embeds: [embed], components: [showRarityButtons()] });
        const collector = sent.createMessageComponentCollector({ time: 60000 });

        collector.on("collect", async (i) => {
          if (i.user.id !== userId) return;

          if (i.customId.startsWith("pack_mythic") || i.customId.startsWith("pack_legendary") || 
              i.customId.startsWith("pack_epic") || i.customId.startsWith("pack_rare") || i.customId.startsWith("pack_common")) {
            
            const rarity = i.customId.split("_")[1].charAt(0).toUpperCase() + i.customId.split("_")[1].slice(1);
            selectedRarity = rarity;
            packCost = getPackCost(rarity);
            const pool = CARD_DATA.filter(c => c.rarity === rarity);
            selectedCard = pool[Math.floor(Math.random() * pool.length)];

            const cardEmbed = new EmbedBuilder()
              .setColor(getRarityColor(rarity))
              .setTitle("📦 Your Pack")
              .setDescription(`**${selectedCard.name}**\n⭐ ${selectedCard.rating}\n🎯 ${selectedCard.pos}\n💎 ${selectedCard.rarity}\n\n💰 Cost: ${packCost.toLocaleString()} coins`)
              .setImage(selectedCard.img);

            i.update({ embeds: [cardEmbed], components: [showOpenButton()] });
          }

          if (i.customId.startsWith("pack_open")) {
            if (!selectedCard) return;
            
            if (user.coins < packCost) {
              return i.reply({ content: `❌ You need ${packCost.toLocaleString()} coins! You have ${user.coins.toLocaleString()} coins.`, ephemeral: true });
            }
            
            user.coins -= packCost;
            user.inventory.push({ name: selectedCard.name, rating: selectedCard.rating, pos: selectedCard.pos, rarity: selectedCard.rarity, img: selectedCard.img });
            await saveUser(userId);
            
            const openEmbed = new EmbedBuilder()
              .setColor(getRarityColor(selectedCard.rarity))
              .setTitle("📦 Pack Recieved!")
              .setDescription(`You got **${selectedCard.name}**\n⭐ ${selectedCard.rating}\n💎 ${selectedCard.rarity}`)
              .setImage(selectedCard.img);

            i.update({ embeds: [openEmbed], components: [] });
            storage.createLog({ level: "info", message: `${message.author.tag} claimed a card: ${selectedCard.name} (${selectedCard.rarity})` });
            collector.stop();
          }

          if (i.customId.startsWith("pack_back")) {
            const embed = new EmbedBuilder()
              .setColor(0xFFD700)
              .setTitle("🏆 FUT Champions Pack")
              .setDescription("Choose which rarity pack you want to open!")
              .setImage("https://cdn.discordapp.com/attachments/1454220184008261736/1454235309825982749/IMG_3159.png");
            
            i.update({ embeds: [embed], components: [showRarityButtons()] });
            selectedRarity = "";
            selectedCard = null;
            packCost = 0;
          }
        });
      }

      // TEAM VIEW - Visual formation image
      if (cmd === "team" && !args[0]) {
        if (Object.keys(user.team).length === 0) {
          return await message.reply("Your team is empty.\n\n**Soccer Positions (1-11):**\n1 GK - Goalkeeper\n2 LB - Left Back\n3 CB - Center Back\n4 CB - Center Back\n5 RB - Right Back\n6 DM - Defensive Mid\n7 CM - Center Mid\n8 CM - Center Mid\n9 LW - Left Wing\n10 ST - Striker\n11 RW - Right Wing\n\nUse `!teamadd <player> <1-11>`");
        }

        // Calculate chemistry for display
        const posMap: Record<string, string[]> = {
          "1": ["GK"], "2": ["DF", "RB", "LB", "CB"], "3": ["DF", "CB", "RB", "LB"], "4": ["DF", "CB"],
          "5": ["DF", "LB", "RB", "CB"], "6": ["CM", "CDM", "CAM"], "7": ["CM", "RM", "CAM", "RW"],
          "8": ["CM", "CDM", "CAM"], "9": ["ST", "CF", "CAM"], "10": ["CAM", "CM", "CF", "ST"],
          "11": ["ST", "LW", "CF", "RW"]
        };
        let chem = 0;
        for (const [pos, card] of Object.entries(user.team)) {
          const validPos = posMap[pos] || [];
          if (validPos.includes(card.pos)) {
            chem += 10;
          } else if (card.pos === "CM" || card.pos === "DF" || card.pos === "ST") {
            chem += 5;
          } else {
            chem += 2;
          }
          chem += Math.floor(card.rating / 20);
        }
        chem = Math.min(chem, 100);

        try {
          const imageBuffer = await generateTeamImage(message.author.username, user.team, chem);
          const attachment = new AttachmentBuilder(imageBuffer, { name: "team.png" });
          
          const totalRating = Object.values(user.team).reduce((sum, c) => sum + c.rating, 0);
          
          const embed = new EmbedBuilder()
            .setColor(0x2ecc71)
            .setTitle(`${message.author.username}'s Team`)
            .setImage("attachment://team.png")
            .setFooter({ text: `Total Rating: ${totalRating} | Chemistry: ${chem} | Players: ${Object.keys(user.team).length}/11` });
          
          await message.reply({ embeds: [embed], files: [attachment] });
        } catch (err) {
          // Fallback to text if image generation fails
          const positions = ["GK", "LB", "CB", "CB", "RB", "DM", "CM", "CM", "LW", "ST", "RW"];
          let teamDesc = "";
          for (let i = 1; i <= 11; i++) {
            const card = user.team[i.toString()];
            const posName = positions[i - 1];
            if (card) {
              teamDesc += `${i}. ${posName}: **${card.name}** (${card.rating})\n`;
            } else {
              teamDesc += `${i}. ${posName}: *Empty*\n`;
            }
          }
          const totalRating = Object.values(user.team).reduce((sum, c) => sum + c.rating, 0);
          const embed = new EmbedBuilder()
            .setColor(0x3498db)
            .setTitle("Your Team Formation")
            .setDescription(teamDesc)
            .setFooter({ text: `Total Rating: ${totalRating} | Players: ${Object.keys(user.team).length}/11` });
          await message.reply({ embeds: [embed] });
        }
      }

      // TEAMADD
      if (cmd === "teamadd") {
        const posNum = args[args.length - 1];
        const playerName = args.slice(0, -1).join(" ");

        if (!posNum || isNaN(parseInt(posNum)) || parseInt(posNum) < 1 || parseInt(posNum) > 11) {
          return await message.reply("❌ Position must be 1-11");
        }

        const names = user.inventory.map(c => c.name);
        const match = findBestMatch(playerName, names).bestMatch?.target;
        const card = user.inventory.find(c => c.name === match);
        if (!card) return await message.reply("❌ You don't own this player.");

        user.team[posNum] = card;
        await saveUser(userId);
        await message.reply(`✅ Added **${card.name}** to position **${posNum}**`);
        storage.createLog({ level: "info", message: `${message.author.tag} added ${card.name} to position ${posNum}` });
      }

      // TEAMREMOVE
      if (cmd === "teamremove") {
        const posNum = args[0];

        if (!posNum || isNaN(parseInt(posNum)) || parseInt(posNum) < 1 || parseInt(posNum) > 11) {
          return await message.reply("❌ Position must be 1-11");
        }

        if (!user.team[posNum]) {
          return await message.reply("❌ Position is empty");
        }

        const removed = user.team[posNum];
        delete user.team[posNum];
        await saveUser(userId);
        await message.reply(`✅ Removed **${removed.name}** from position **${posNum}**`);
        storage.createLog({ level: "info", message: `${message.author.tag} removed ${removed.name} from position ${posNum}` });
      }

      // SWAP
      if (cmd === "swap") {
        const pos1 = args[0];
        const pos2 = args[1];

        if (!pos1 || !pos2 || isNaN(parseInt(pos1)) || isNaN(parseInt(pos2)) || 
            parseInt(pos1) < 1 || parseInt(pos1) > 11 || parseInt(pos2) < 1 || parseInt(pos2) > 11) {
          return await message.reply("❌ Use `!swap <pos1> <pos2>` where both are 1-11");
        }

        if (!user.team[pos1]) {
          return await message.reply(`❌ Position **${pos1}** is empty`);
        }

        if (!user.team[pos2]) {
          return await message.reply(`❌ Position **${pos2}** is empty`);
        }

        const card1 = user.team[pos1];
        const card2 = user.team[pos2];

        user.team[pos1] = card2;
        user.team[pos2] = card1;
        await saveUser(userId);

        await message.reply(`✅ Swapped **${card1.name}** (pos ${pos1}) ↔ **${card2.name}** (pos ${pos2})`);
        storage.createLog({ level: "info", message: `${message.author.tag} swapped positions ${pos1} and ${pos2}` });
      }

      // FRIENDLY - Live match with phases
      if (cmd === "friendly") {
        const opponent = message.mentions.users.first();
        if (!opponent) return await message.reply("Mention someone to challenge");
        if (opponent.id === userId) return await message.reply("You can't play against yourself");

        const oppUser = await getUser(opponent.id);

        const teamCount1 = Object.keys(user.team).length;
        const teamCount2 = Object.keys(oppUser.team).length;
        
        if (teamCount1 < 11) {
          return await message.reply(`You need 11 players in your team. You have ${teamCount1}/11. Use \`!teamadd <player> <1-11>\``);
        }
        if (teamCount2 < 11) {
          return await message.reply(`Your opponent needs 11 players. They have ${teamCount2}/11.`);
        }

        // Calculate chemistry (based on position fit and rating)
        function calculateChemistry(team: Record<string, BotCard>): number {
          let chem = 0;
          const posMap: Record<string, string[]> = {
            "1": ["GK"], "2": ["DF", "RB", "LB", "CB"], "3": ["DF", "CB", "RB", "LB"], "4": ["DF", "CB"],
            "5": ["DF", "LB", "RB", "CB"], "6": ["CM", "CDM", "CAM"], "7": ["CM", "RM", "CAM", "RW"],
            "8": ["CM", "CDM", "CAM"], "9": ["ST", "CF", "CAM"], "10": ["CAM", "CM", "CF", "ST"],
            "11": ["ST", "LW", "CF", "RW"]
          };
          for (const [pos, card] of Object.entries(team)) {
            const validPos = posMap[pos] || [];
            if (validPos.includes(card.pos)) {
              chem += 10;
            } else if (card.pos === "CM" || card.pos === "DF" || card.pos === "ST") {
              chem += 5;
            } else {
              chem += 2;
            }
            // Rating bonus
            chem += Math.floor(card.rating / 20);
          }
          return Math.min(chem, 100);
        }

        const chem1 = calculateChemistry(user.team);
        const chem2 = calculateChemistry(oppUser.team);
        const power1 = Object.values(user.team).reduce((sum, c) => sum + c.rating, 0);
        const power2 = Object.values(oppUser.team).reduce((sum, c) => sum + c.rating, 0);

        // Live match simulation
        const team1Players = Object.values(user.team);
        const team2Players = Object.values(oppUser.team);
        
        interface MatchEvent {
          minute: number;
          type: "goal" | "yellow" | "red" | "penalty";
          team: 1 | 2;
          player: string;
          assist?: string;
          penaltyScored?: boolean;
        }
        
        const events: MatchEvent[] = [];
        let score1 = 0, score2 = 0;
        
        // Generate events for each half
        function generateHalfEvents(startMin: number, endMin: number) {
          for (let min = startMin; min <= endMin; min++) {
            // Goal chance based on power and chemistry
            const goalChance1 = (power1 / 1000 + chem1 / 100) * 0.015;
            const goalChance2 = (power2 / 1000 + chem2 / 100) * 0.015;
            
            if (Math.random() < goalChance1) {
              const attackers = team1Players.filter(p => ["ST", "CF", "CAM", "RW", "LW"].includes(p.pos));
              const midfielders = team1Players.filter(p => ["CM", "CDM", "RM", "LM"].includes(p.pos));
              const scorer = attackers.length > 0 ? attackers[Math.floor(Math.random() * attackers.length)] : team1Players[Math.floor(Math.random() * team1Players.length)];
              const assister = midfielders.length > 0 && Math.random() > 0.3 ? midfielders[Math.floor(Math.random() * midfielders.length)] : undefined;
              events.push({ minute: min, type: "goal", team: 1, player: scorer.name, assist: assister?.name });
              score1++;
            }
            
            if (Math.random() < goalChance2) {
              const attackers = team2Players.filter(p => ["ST", "CF", "CAM", "RW", "LW"].includes(p.pos));
              const midfielders = team2Players.filter(p => ["CM", "CDM", "RM", "LM"].includes(p.pos));
              const scorer = attackers.length > 0 ? attackers[Math.floor(Math.random() * attackers.length)] : team2Players[Math.floor(Math.random() * team2Players.length)];
              const assister = midfielders.length > 0 && Math.random() > 0.3 ? midfielders[Math.floor(Math.random() * midfielders.length)] : undefined;
              events.push({ minute: min, type: "goal", team: 2, player: scorer.name, assist: assister?.name });
              score2++;
            }
            
            // Yellow card (very low probability ~2%)
            if (Math.random() < 0.02) {
              const team = Math.random() > 0.5 ? 1 : 2;
              const players = team === 1 ? team1Players : team2Players;
              const defender = players.filter(p => ["DF", "CB", "LB", "RB", "CDM"].includes(p.pos));
              const player = defender.length > 0 ? defender[Math.floor(Math.random() * defender.length)] : players[Math.floor(Math.random() * players.length)];
              events.push({ minute: min, type: "yellow", team, player: player.name });
            }
            
            // Red card (very rare ~0.5%)
            if (Math.random() < 0.005) {
              const team = Math.random() > 0.5 ? 1 : 2;
              const players = team === 1 ? team1Players : team2Players;
              const player = players[Math.floor(Math.random() * players.length)];
              events.push({ minute: min, type: "red", team, player: player.name });
            }

            // Penalty (rare ~0.8% chance per minute)
            if (Math.random() < 0.008) {
              const team = Math.random() > 0.5 ? 1 : 2;
              const shootingTeam = team === 1 ? team1Players : team2Players;
              const defendingTeam = team === 1 ? team2Players : team1Players;
              
              // Get penalty taker (usually attackers)
              const attackers = shootingTeam.filter(p => ["ST", "CF", "CAM"].includes(p.pos));
              const shooter = attackers.length > 0 ? attackers[Math.floor(Math.random() * attackers.length)] : shootingTeam[Math.floor(Math.random() * shootingTeam.length)];
              
              // Get keeper
              const keeper = defendingTeam.find(p => p.pos === "GK") || defendingTeam[0];
              
              // Calculate if penalty is scored (based on keeper rating)
              // Higher keeper rating = lower chance to score
              const keeperRating = keeper.rating;
              const saveChance = (keeperRating - 70) / 100; // 70 rating = 0%, 90 rating = 20%, 97 rating = 27%
              const scored = Math.random() > saveChance;
              
              events.push({ 
                minute: min, 
                type: "penalty", 
                team, 
                player: shooter.name,
                assist: keeper.name,
                penaltyScored: scored
              });
              
              if (scored) {
                if (team === 1) score1++;
                else score2++;
              }
            }
          }
        }

        // First half
        generateHalfEvents(1, 45);
        const halfTimeScore1 = score1;
        const halfTimeScore2 = score2;
        
        // Second half
        generateHalfEvents(46, 90);

        // Sort events by minute
        events.sort((a, b) => a.minute - b.minute);

        // Build match display
        const formatEvent = (e: MatchEvent) => {
          const teamName = e.team === 1 ? message.author.username : opponent.username;
          if (e.type === "goal") {
            return `**${e.minute}'** ${e.team === 1 ? "<<" : ">>"} GOAL! ${e.player}${e.assist ? ` (assist: ${e.assist})` : ""}`;
          } else if (e.type === "yellow") {
            return `**${e.minute}'** Yellow card: ${e.player} (${teamName})`;
          } else if (e.type === "red") {
            return `**${e.minute}'** RED CARD: ${e.player} (${teamName})`;
          } else if (e.type === "penalty") {
            return `**${e.minute}'** PENALTY! ${e.player} vs ${e.assist} - ${e.penaltyScored ? "GOAL!" : "SAVED!"}`;
          }
          return "";
        };

        const firstHalfEvents = events.filter(e => e.minute <= 45);
        const secondHalfEvents = events.filter(e => e.minute > 45);

        // Send kick-off message
        const kickoffEmbed = new EmbedBuilder()
          .setColor(0x2ecc71)
          .setTitle("KICK-OFF!")
          .setDescription(`**${message.author.username}** (Chem: ${chem1}) vs **${opponent.username}** (Chem: ${chem2})\n\nMatch starting...`);
        
        const matchMsg = await message.reply({ embeds: [kickoffEmbed] });
        await new Promise(r => setTimeout(r, 2000));

        // FIRST HALF - Minute by minute (1-45), 2 seconds per minute
        let displayedEvents: string[] = [];
        let runningScore1 = 0;
        let runningScore2 = 0;

        for (let min = 1; min <= 45; min++) {
          // Check if any events happened this minute
          const minEvents = events.filter(e => e.minute === min);
          for (const e of minEvents) {
            if (e.type === "goal") {
              if (e.team === 1) runningScore1++;
              else runningScore2++;
            }
            if (e.type === "penalty" && e.penaltyScored) {
              if (e.team === 1) runningScore1++;
              else runningScore2++;
            }
            displayedEvents.push(formatEvent(e));

            // Show penalty image if it's a penalty
            if (e.type === "penalty") {
              try {
                const shootDir = PENALTY_DIRECTIONS[Math.floor(Math.random() * PENALTY_DIRECTIONS.length)];
                const keeperDir = PENALTY_DIRECTIONS[Math.floor(Math.random() * PENALTY_DIRECTIONS.length)];
                const penaltyImage = await generatePenaltyImage(
                  shootDir, 
                  keeperDir, 
                  e.penaltyScored || false, 
                  e.player, 
                  e.assist || "Keeper"
                );
                const penaltyAttachment = new AttachmentBuilder(penaltyImage, { name: "penalty.png" });
                const penaltyEmbed = new EmbedBuilder()
                  .setColor(e.penaltyScored ? 0x00ff00 : 0xff0000)
                  .setTitle(`${min}' - PENALTY!`)
                  .setImage("attachment://penalty.png")
                  .setDescription(`**${message.author.username}** ${runningScore1} - ${runningScore2} **${opponent.username}**`);
                await matchMsg.edit({ embeds: [penaltyEmbed], files: [penaltyAttachment] });
                await new Promise(r => setTimeout(r, 3000));
              } catch (err) {
                // Continue if image fails
              }
            }
          }

          const firstHalfEmbed = new EmbedBuilder()
            .setColor(0xf1c40f)
            .setTitle(`FIRST HALF - ${min}'`)
            .setDescription(
              `**${message.author.username}** ${runningScore1} - ${runningScore2} **${opponent.username}**\n\n` +
              (displayedEvents.length > 0 ? displayedEvents.join("\n") : "No events yet...")
            );
          
          await matchMsg.edit({ embeds: [firstHalfEmbed], files: [] });
          await new Promise(r => setTimeout(r, 1000));
        }

        // Halftime break (10 seconds)
        const halftimeEmbed = new EmbedBuilder()
          .setColor(0x3498db)
          .setTitle("HALFTIME")
          .setDescription(`**${message.author.username}** ${halfTimeScore1} - ${halfTimeScore2} **${opponent.username}**\n\n10 second break...\n\n` +
            (displayedEvents.length > 0 ? "**First Half Events:**\n" + displayedEvents.join("\n") : "No events in first half"));
        
        await matchMsg.edit({ embeds: [halftimeEmbed] });
        await new Promise(r => setTimeout(r, 10000));

        // SECOND HALF - Minute by minute (46-90), 2 seconds per minute
        let secondHalfDisplayedEvents: string[] = [];

        for (let min = 46; min <= 90; min++) {
          // Check if any events happened this minute
          const minEvents = events.filter(e => e.minute === min);
          for (const e of minEvents) {
            if (e.type === "goal") {
              if (e.team === 1) runningScore1++;
              else runningScore2++;
            }
            if (e.type === "penalty" && e.penaltyScored) {
              if (e.team === 1) runningScore1++;
              else runningScore2++;
            }
            secondHalfDisplayedEvents.push(formatEvent(e));

            // Show penalty image if it's a penalty
            if (e.type === "penalty") {
              try {
                const shootDir = PENALTY_DIRECTIONS[Math.floor(Math.random() * PENALTY_DIRECTIONS.length)];
                const keeperDir = PENALTY_DIRECTIONS[Math.floor(Math.random() * PENALTY_DIRECTIONS.length)];
                const penaltyImage = await generatePenaltyImage(
                  shootDir, 
                  keeperDir, 
                  e.penaltyScored || false, 
                  e.player, 
                  e.assist || "Keeper"
                );
                const penaltyAttachment = new AttachmentBuilder(penaltyImage, { name: "penalty.png" });
                const penaltyEmbed = new EmbedBuilder()
                  .setColor(e.penaltyScored ? 0x00ff00 : 0xff0000)
                  .setTitle(`${min}' - PENALTY!`)
                  .setImage("attachment://penalty.png")
                  .setDescription(`**${message.author.username}** ${runningScore1} - ${runningScore2} **${opponent.username}**`);
                await matchMsg.edit({ embeds: [penaltyEmbed], files: [penaltyAttachment] });
                await new Promise(r => setTimeout(r, 3000));
              } catch (err) {
                // Continue if image fails
              }
            }
          }

          const secondHalfEmbed = new EmbedBuilder()
            .setColor(0xe67e22)
            .setTitle(`SECOND HALF - ${min}'`)
            .setDescription(
              `**${message.author.username}** ${runningScore1} - ${runningScore2} **${opponent.username}**\n\n` +
              (secondHalfDisplayedEvents.length > 0 ? secondHalfDisplayedEvents.join("\n") : "No events yet...")
            );
          
          await matchMsg.edit({ embeds: [secondHalfEmbed], files: [] });
          await new Promise(r => setTimeout(r, 1000));
        }

        // Final result
        let resultText = "";
        if (score1 > score2) {
          user.coins += 5000;
          await saveUser(userId);
          resultText = `${message.author.username} WINS! +5,000 coins`;
        } else if (score2 > score1) {
          oppUser.coins += 5000;
          await saveUser(opponent.id);
          resultText = `${opponent.username} WINS! +5,000 coins`;
        } else {
          resultText = "DRAW!";
        }

        // Collect scorers and assists
        const scorersList: string[] = [];
        const assistsList: string[] = [];
        const yellowCards: string[] = [];
        const redCards: string[] = [];
        
        events.forEach(e => {
          if (e.type === "goal") {
            scorersList.push(`${e.player} (${e.team === 1 ? message.author.username : opponent.username})`);
            if (e.assist) assistsList.push(`${e.assist} (${e.team === 1 ? message.author.username : opponent.username})`);
          }
          if (e.type === "yellow") yellowCards.push(`${e.player} (${e.team === 1 ? message.author.username : opponent.username})`);
          if (e.type === "red") redCards.push(`${e.player} (${e.team === 1 ? message.author.username : opponent.username})`);
        });

        const finalEmbed = new EmbedBuilder()
          .setColor(score1 > score2 ? 0x2ecc71 : score2 > score1 ? 0xe74c3c : 0x95a5a6)
          .setTitle("FULL TIME")
          .setDescription(
            `**${message.author.username}** ${score1} - ${score2} **${opponent.username}**\n\n` +
            `${resultText}\n\n` +
            (scorersList.length > 0 ? `**Scorers:** ${scorersList.join(", ")}\n` : "") +
            (assistsList.length > 0 ? `**Assists:** ${assistsList.join(", ")}\n` : "") +
            (yellowCards.length > 0 ? `**Yellow Cards:** ${yellowCards.join(", ")}\n` : "") +
            (redCards.length > 0 ? `**Red Cards:** ${redCards.join(", ")}` : "")
          )
          .setFooter({ text: `Chemistry: ${message.author.username} ${chem1} | ${opponent.username} ${chem2}` });

        await matchMsg.edit({ embeds: [finalEmbed] });

        // Save match result to database
        await storage.saveMatchResult({
          player1Id: userId,
          player1Name: message.author.username,
          player2Id: opponent.id,
          player2Name: opponent.username,
          player1Goals: score1,
          player2Goals: score2,
          player1Chemistry: chem1,
          player2Chemistry: chem2,
          scorers: JSON.stringify(scorersList),
          assists: JSON.stringify(assistsList),
          cards: JSON.stringify([...yellowCards.map(c => ({ type: "yellow", player: c })), ...redCards.map(c => ({ type: "red", player: c }))])
        });

        // Update player stats
        const playerStats: Record<string, { goals: number; assists: number; yellow: number; red: number }> = {};
        events.forEach(e => {
          if (!playerStats[e.player]) playerStats[e.player] = { goals: 0, assists: 0, yellow: 0, red: 0 };
          if (e.type === "goal") {
            playerStats[e.player].goals++;
            if (e.assist) {
              if (!playerStats[e.assist]) playerStats[e.assist] = { goals: 0, assists: 0, yellow: 0, red: 0 };
              playerStats[e.assist].assists++;
            }
          }
          if (e.type === "yellow") playerStats[e.player].yellow++;
          if (e.type === "red") playerStats[e.player].red++;
        });

        // Update all players stats (including those who played but no events)
        const allPlayers = [...team1Players.map(p => p.name), ...team2Players.map(p => p.name)];
        const uniquePlayers = [...new Set(allPlayers)];
        for (const playerName of uniquePlayers) {
          const stats = playerStats[playerName] || { goals: 0, assists: 0, yellow: 0, red: 0 };
          await storage.updatePlayerStats(playerName, stats.goals, stats.assists, stats.yellow, stats.red);
        }

        storage.createLog({ level: "info", message: `${message.author.tag} played friendly match vs ${opponent.tag}: ${score1}-${score2}` });
      }

      // SHOW - Display player stats
      if (cmd === "show") {
        const playerName = args.join(" ");
        if (!playerName) {
          return await message.reply("Use `!show <player name>` to view player statistics");
        }

        const names = CARD_DATA.map(c => c.name);
        const match = findBestMatch(playerName, names).bestMatch?.target;
        if (!match) {
          return await message.reply("Player not found");
        }

        const stats = await storage.getPlayerStats(match);
        const card = CARD_DATA.find(c => c.name === match);

        const embed = new EmbedBuilder()
          .setColor(card ? getRarityColor(card.rarity) : 0x3498db)
          .setTitle(`Player Stats: ${match}`)
          .setDescription(
            stats ? 
            `**Matches Played:** ${stats.matches}\n` +
            `**Goals:** ${stats.goals}\n` +
            `**Assists:** ${stats.assists}\n` +
            `**Yellow Cards:** ${stats.yellowCards}\n` +
            `**Red Cards:** ${stats.redCards}` :
            "No match data yet for this player"
          );

        if (card) {
          embed.setThumbnail(card.img);
        }

        await message.reply({ embeds: [embed] });
      }

      // RESULT - Show match history
      if (cmd === "result" || cmd === "results") {
        const results = await storage.getMatchResults(10);
        
        if (results.length === 0) {
          return await message.reply("No matches played yet");
        }

        const desc = results.map((r, i) => {
          const winner = r.player1Goals > r.player2Goals ? r.player1Name : 
                         r.player2Goals > r.player1Goals ? r.player2Name : "Draw";
          return `**${i + 1}.** ${r.player1Name} ${r.player1Goals} - ${r.player2Goals} ${r.player2Name} | ${winner === "Draw" ? "Draw" : `Winner: ${winner}`}`;
        }).join("\n");

        const embed = new EmbedBuilder()
          .setColor(0x3498db)
          .setTitle("Recent Match Results")
          .setDescription(desc);

        await message.reply({ embeds: [embed] });
      }

      // ALL CARDS
      if (cmd === "all") {
        const mythic = CARD_DATA.filter(c => c.rarity === "Mythic");
        const legendary = CARD_DATA.filter(c => c.rarity === "Legendary");
        const epic = CARD_DATA.filter(c => c.rarity === "Epic");
        const rare = CARD_DATA.filter(c => c.rarity === "Rare");
        const common = CARD_DATA.filter(c => c.rarity === "Common");

        const embeds: EmbedBuilder[] = [];
        
        if (mythic.length > 0) {
          const desc = mythic.map(c => `• ${c.name} – ${c.rating} – ${c.pos} – 💰 ${priceFromRating(c.rating).toLocaleString()} coins`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0xFF6B00).setTitle("⭐ MYTHIC (96–99)").setDescription(desc));
        }
        
        if (legendary.length > 0) {
          const desc = legendary.map(c => `• ${c.name} – ${c.rating} – ${c.pos} – 💰 ${priceFromRating(c.rating).toLocaleString()} coins`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0xFFD700).setTitle("⭐ LEGENDARY (90–95)").setDescription(desc));
        }
        
        if (epic.length > 0) {
          const desc = epic.map(c => `• ${c.name} – ${c.rating} – ${c.pos} – 💰 ${priceFromRating(c.rating).toLocaleString()} coins`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x9D4EDD).setTitle("⭐ EPIC (87–89)").setDescription(desc));
        }
        
        if (rare.length > 0) {
          const desc = rare.map(c => `• ${c.name} – ${c.rating} – ${c.pos} – 💰 ${priceFromRating(c.rating).toLocaleString()} coins`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x00BFFF).setTitle("⭐ RARE (83–86)").setDescription(desc));
        }
        
        if (common.length > 0) {
          const desc = common.map(c => `• ${c.name} – ${c.rating} – ${c.pos} – 💰 ${priceFromRating(c.rating).toLocaleString()} coins`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x888888).setTitle("⭐ COMMON (60–82)").setDescription(desc));
        }

        embeds.push(new EmbedBuilder().setColor(0x3498db).setDescription("Use **/claim** or mention the bot to pull a random card"));

        await message.reply({ embeds });
        storage.createLog({ level: "info", message: `${message.author.tag} viewed all cards` });
      }
    } catch (error) {
      console.error("Command error:", error);
      storage.createLog({ level: "error", message: `Command error: ${error}` });
    }
  });

  // Handle slash commands
  client.on(Events.InteractionCreate, async (interaction) => {
    if (!interaction.isChatInputCommand()) return;

    const userId = interaction.user.id;
    const user = await getUser(userId);
    const cmd = interaction.commandName;

    try {
      // CLAIM via slash command
      if (cmd === "claim") {
        const now = Date.now();
        if (now < user.cooldown) {
          return await interaction.reply({ content: `Claim again <t:${Math.floor(user.cooldown / 1000)}:R>`, ephemeral: true });
        }

        user.cooldown = now + 3600000;
        await saveUser(userId);

        const rarity = rarityRoll();
        const pool = CARD_DATA.filter(c => c.rarity === rarity);
        const card = pool[Math.floor(Math.random() * pool.length)];

        const cardImage = await generateCardImage(card);
        const attachment = new AttachmentBuilder(cardImage, { name: "card.png" });

        const embed = new EmbedBuilder()
          .setColor(getRarityColor(rarity))
          .setTitle("Football Card Claimed")
          .setDescription(`**${card.name}**\nRating: ${card.rating}\nPosition: ${card.pos}\nRarity: ${card.rarity}`)
          .setImage("attachment://card.png");

        const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
          new ButtonBuilder().setCustomId(`keep_${userId}`).setLabel("Keep").setStyle(ButtonStyle.Success),
          new ButtonBuilder().setCustomId(`sell_${userId}`).setLabel("Sell").setStyle(ButtonStyle.Danger)
        );

        const reply = await interaction.reply({ embeds: [embed], files: [attachment], components: [row], fetchReply: true });
        const collector = reply.createMessageComponentCollector({ time: 60000 });

        collector.on("collect", async (i) => {
          if (i.user.id !== userId) return;

          if (i.customId === `keep_${userId}`) {
            user.inventory.push({ name: card.name, rating: card.rating, pos: card.pos, rarity: card.rarity, img: card.img });
            await saveUser(userId);
            i.update({ content: "Card added to inventory", embeds: [], components: [], files: [] });
            storage.createLog({ level: "info", message: `${interaction.user.tag} kept card: ${card.name}` });
          }

          if (i.customId === `sell_${userId}`) {
            const sellPrice = Math.floor(priceFromRating(card.rating) / 2);
            user.coins += sellPrice;
            await saveUser(userId);
            i.update({ content: `Sold for ${sellPrice.toLocaleString()} coins`, embeds: [], components: [], files: [] });
            storage.createLog({ level: "info", message: `${interaction.user.tag} sold card: ${card.name}` });
          }

          collector.stop();
        });

        storage.createLog({ level: "info", message: `${interaction.user.tag} claimed a card: ${card.name}` });
      }

      // COINS
      if (cmd === "coins") {
        await interaction.reply(`You have ${user.coins.toLocaleString()} coins`);
      }

      // CLUB
      if (cmd === "club" || cmd === "inv") {
        if (user.inventory.length === 0) {
          return await interaction.reply("Your club is empty.");
        }
        const inv = user.inventory.map(c => `${c.name} (${c.rating})`).join("\n");
        await interaction.reply(`**Club**\n${inv}\n\n**Balance**: ${user.coins.toLocaleString()} coins`);
        storage.createLog({ level: "info", message: `${interaction.user.tag} viewed club` });
      }

      // TEAM
      if (cmd === "team") {
        const teamCount = Object.keys(user.team).length;
        if (teamCount === 0) {
          return await interaction.reply("Your team is empty. Use `/teamadd` to add players.");
        }

        const chem = calculateChemistryStatic(user.team);
        const teamImage = await generateTeamImage(interaction.user.username, user.team, chem);
        const attachment = new AttachmentBuilder(teamImage, { name: "team.png" });

        await interaction.reply({ files: [attachment] });
        storage.createLog({ level: "info", message: `${interaction.user.tag} viewed team` });
      }

      // SHOW
      if (cmd === "show") {
        const name = interaction.options.getString("player") || "";
        const names = user.inventory.map(c => c.name);
        const match = findBestMatch(name, names).bestMatch?.target;
        const card = user.inventory.find(c => c.name === match);
        if (!card) return await interaction.reply({ content: "You don't own this card.", ephemeral: true });

        const stats = user.stats[card.name] || { goals: 0, assists: 0, yellowCards: 0, redCards: 0, matches: 0 };
        const price = priceFromRating(card.rating);
        
        const cardImage = await generateCardImage(card, stats);
        const attachment = new AttachmentBuilder(cardImage, { name: "card.png" });

        const embed = new EmbedBuilder()
          .setColor(getRarityColor(card.rarity))
          .setTitle(`${card.name}`)
          .setDescription(`**Rating**: ${card.rating}\n**Position**: ${card.pos}\n**Rarity**: ${card.rarity}\n**Worth**: ${price.toLocaleString()} coins`)
          .setImage("attachment://card.png");

        await interaction.reply({ embeds: [embed], files: [attachment] });
      }

      // TEAMADD
      if (cmd === "teamadd") {
        const playerName = interaction.options.getString("player") || "";
        const position = interaction.options.getInteger("position") || 0;

        const names = user.inventory.map(c => c.name);
        const match = findBestMatch(playerName, names).bestMatch?.target;
        const card = user.inventory.find(c => c.name === match);

        if (!card) return await interaction.reply({ content: "You don't own this card.", ephemeral: true });

        user.team[String(position)] = card;
        await saveUser(userId);

        await interaction.reply(`Added **${card.name}** to position ${position}`);
        storage.createLog({ level: "info", message: `${interaction.user.tag} added ${card.name} to position ${position}` });
      }

      // TEAMREMOVE
      if (cmd === "teamremove") {
        const position = interaction.options.getInteger("position") || 0;
        const posStr = String(position);

        if (!user.team[posStr]) {
          return await interaction.reply({ content: `Position ${position} is empty`, ephemeral: true });
        }

        const removed = user.team[posStr];
        delete user.team[posStr];
        await saveUser(userId);

        await interaction.reply(`Removed **${removed.name}** from position ${position}`);
        storage.createLog({ level: "info", message: `${interaction.user.tag} removed ${removed.name} from position ${position}` });
      }

      // SWAP
      if (cmd === "swap") {
        const pos1 = String(interaction.options.getInteger("pos1") || 0);
        const pos2 = String(interaction.options.getInteger("pos2") || 0);

        if (!user.team[pos1]) {
          return await interaction.reply({ content: `Position ${pos1} is empty`, ephemeral: true });
        }
        if (!user.team[pos2]) {
          return await interaction.reply({ content: `Position ${pos2} is empty`, ephemeral: true });
        }

        const card1 = user.team[pos1];
        const card2 = user.team[pos2];
        user.team[pos1] = card2;
        user.team[pos2] = card1;
        await saveUser(userId);

        await interaction.reply(`Swapped **${card1.name}** (pos ${pos1}) with **${card2.name}** (pos ${pos2})`);
        storage.createLog({ level: "info", message: `${interaction.user.tag} swapped positions ${pos1} and ${pos2}` });
      }

      // ALL
      if (cmd === "all") {
        const mythic = CARD_DATA.filter(c => c.rarity === "Mythic");
        const legendary = CARD_DATA.filter(c => c.rarity === "Legendary");
        const epic = CARD_DATA.filter(c => c.rarity === "Epic");
        const rare = CARD_DATA.filter(c => c.rarity === "Rare");
        const common = CARD_DATA.filter(c => c.rarity === "Common");

        const embeds: EmbedBuilder[] = [];
        
        if (mythic.length > 0) {
          const desc = mythic.map(c => `${c.name} - ${c.rating} - ${c.pos}`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0xFF6B00).setTitle("MYTHIC").setDescription(desc));
        }
        if (legendary.length > 0) {
          const desc = legendary.map(c => `${c.name} - ${c.rating} - ${c.pos}`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0xFFD700).setTitle("LEGENDARY").setDescription(desc));
        }
        if (epic.length > 0) {
          const desc = epic.map(c => `${c.name} - ${c.rating} - ${c.pos}`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x9D4EDD).setTitle("EPIC").setDescription(desc));
        }
        if (rare.length > 0) {
          const desc = rare.map(c => `${c.name} - ${c.rating} - ${c.pos}`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x00BFFF).setTitle("RARE").setDescription(desc));
        }
        if (common.length > 0) {
          const desc = common.map(c => `${c.name} - ${c.rating} - ${c.pos}`).join("\n");
          embeds.push(new EmbedBuilder().setColor(0x888888).setTitle("COMMON").setDescription(desc));
        }

        await interaction.reply({ embeds });
        storage.createLog({ level: "info", message: `${interaction.user.tag} viewed all cards` });
      }

      // RESULT
      if (cmd === "result") {
        const results = await storage.getMatchResults(userId);
        if (!results || results.length === 0) {
          return await interaction.reply("No match history found.");
        }

        const desc = results.slice(0, 5).map(r => {
          return `**${r.player1Name}** ${r.player1Goals} - ${r.player2Goals} **${r.player2Name}** (${r.createdAt ? new Date(r.createdAt).toLocaleDateString() : "Unknown"})`;
        }).join("\n");

        const embed = new EmbedBuilder()
          .setColor(0x3498db)
          .setTitle("Recent Match Results")
          .setDescription(desc);

        await interaction.reply({ embeds: [embed] });
      }

    } catch (error) {
      console.error("Slash command error:", error);
      storage.createLog({ level: "error", message: `Slash command error: ${error}` });
      if (!interaction.replied) {
        await interaction.reply({ content: "An error occurred", ephemeral: true });
      }
    }
  });

  try {
    await client.login(process.env.DISCORD_TOKEN);
    await seedCards();
  } catch (error) {
    console.error("Failed to login to Discord:", error);
    storage.createLog({ level: "error", message: `Failed to login: ${error}` });
  }
}

// Static chemistry calculator for slash commands
function calculateChemistryStatic(team: Record<string, BotCard>): number {
  let chem = 0;
  const posMap: Record<string, string[]> = {
    "1": ["GK"], "2": ["DF", "RB", "LB", "CB"], "3": ["DF", "CB", "RB", "LB"], "4": ["DF", "CB"],
    "5": ["DF", "LB", "RB", "CB"], "6": ["CM", "CDM", "CAM"], "7": ["CM", "RM", "CAM", "RW"],
    "8": ["CM", "CDM", "CAM"], "9": ["ST", "CF", "CAM"], "10": ["CAM", "CM", "CF", "ST"],
    "11": ["ST", "LW", "CF", "RW"]
  };
  for (const [pos, card] of Object.entries(team)) {
    const validPos = posMap[pos] || [];
    if (validPos.includes(card.pos)) {
      chem += 10;
    } else if (card.pos === "CM" || card.pos === "DF" || card.pos === "ST") {
      chem += 5;
    } else {
      chem += 2;
    }
    chem += Math.floor(card.rating / 20);
  }
  return Math.min(chem, 100);
}