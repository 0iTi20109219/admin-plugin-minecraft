# admin-plugin-minecraft

package zero.iti.adminPlugin;

import org.bukkit.Bukkit;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.block.BlockBreakEvent;
import org.bukkit.event.player.*;
import org.bukkit.inventory.Inventory;
import org.bukkit.inventory.ItemStack;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitRunnable;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.URL;
import java.util.*;
import java.lang.management.ManagementFactory;
import com.sun.management.OperatingSystemMXBean;

public class AdminPlugin extends JavaPlugin implements Listener {

    private final Map<String, Boolean> mutedPlayers = new HashMap<>();
    private final Map<String, Double> playerSpeeds = new HashMap<>();
    private final Map<UUID, Boolean> invertedControls = new HashMap<>();
    private final Map<UUID, String> pendingTpRequests = new HashMap<>();
    private final Set<String> vpnAddresses = new HashSet<>();
    private long lastAlertTime = 0; // リソースアラートの頻度を抑えるためのタイマー
    private OperatingSystemMXBean osBean; // OS情報を取得するためのBean
    private int blockBreakThreshold = 100; // 荒らし検出のブロック破壊閾値
    private final Map<UUID, Integer> blockBreakCounts = new HashMap<>();
    private final long resourceCheckInterval = 60000; // リソースチェックの頻度（ミリ秒単位）
    private final long alertCooldown = 300000; // アラート通知のクールダウン時間（ミリ秒単位）
    private Random random = new Random();


    @Override
    public void onEnable() {
        getLogger().info("AdminPlugin has been enabled!");
        getServer().getPluginManager().registerEvents(this, this);
        getServer().getPluginManager().registerEvents(new ChatListener(this), this);
        osBean = (OperatingSystemMXBean) ManagementFactory.getOperatingSystemMXBean();

        // VPNブラックリストをロード
        loadVPNBlacklist();

        // リソース監視タイマーを設定
        startResourceMonitorTask();
    }

    @Override
    public void onDisable() {
        getLogger().info("AdminPlugin has been disabled!");
    }

    // --- ヘルパーメソッド ---

    private void loadVPNBlacklist() {
        // ここにVPNブラックリストの取得処理を実装
        // 例: テキストファイルや外部APIからロード
        try {
            URL url = new URL("https://raw.githubusercontent.com/m1n3r4l/vpn-blacklist/main/vpn-blacklist.txt");
            BufferedReader in = new BufferedReader(new InputStreamReader(url.openStream()));
            String line;
            while ((line = in.readLine()) != null) {
                vpnAddresses.add(line.trim());
            }
            in.close();
            getLogger().info("VPN blacklist loaded.");
        } catch (IOException e) {
            getLogger().warning("Failed to load VPN blacklist. " + e.getMessage());
        }
    }

    private boolean isPlayerOnVPN(Player player) {
        try {
            String ipAddress = player.getAddress().getAddress().getHostAddress();
            for(String vpnIp : vpnAddresses){
                if(ipAddress.startsWith(vpnIp)) {
                    return true;
                }
            }

        } catch (Exception e) {
            getLogger().warning("Error checking IP for VPN. " + e.getMessage());
        }
        return false;
    }

    private void startResourceMonitorTask() {
        new BukkitRunnable() {
            @Override
            public void run() {
                checkResources();
            }
        }.runTaskTimer(this, 0, resourceCheckInterval / 50); // 20 ticks per second
    }

    private void checkResources() {
        double memoryUsage = (double) (Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory()) / Runtime.getRuntime().maxMemory() * 100;
        double cpuUsage = osBean.getSystemCpuLoad() * 100;

        long diskSpace = -1;
        try{
            diskSpace =  new java.io.File(".").getUsableSpace();
        } catch(Exception e) {
            getLogger().warning("Could not get usable disk space: " + e.getMessage());
        }


        if (memoryUsage > 90 || cpuUsage > 90 || (diskSpace != -1 && diskSpace / new java.io.File(".").getTotalSpace() * 100 > 90)) {
            if(System.currentTimeMillis() - lastAlertTime > alertCooldown) {
                String message = "§c[Resource Alert] ";
                if (memoryUsage > 90) {
                    message += String.format("Memory Usage: %.2f%%, ", memoryUsage);
                }
                if (cpuUsage > 90) {
                    message += String.format("CPU Usage: %.2f%%, ", cpuUsage);
                }
                if (diskSpace != -1 && diskSpace / new java.io.File(".").getTotalSpace() * 100 > 90) {
                    message += String.format("Disk Space: %.2f%%, ", (double)diskSpace / new java.io.File(".").getTotalSpace() * 100);
                }

                Bukkit.broadcastMessage(message);
                lastAlertTime = System.currentTimeMillis();
            }
        }
    }


    public boolean isMuted(String playerName) {
        return mutedPlayers.getOrDefault(playerName, false);
    }

    public void setMuted(String playerName, boolean muted) {
        mutedPlayers.put(playerName, muted);
    }

    @EventHandler
    public void onPlayerJoin(PlayerJoinEvent event) {
        Player player = event.getPlayer();
        String ipAddress = player.getAddress().getAddress().getHostAddress();

        // VPN check
        if (isPlayerOnVPN(player)) {
            player.kickPlayer("§cVPN usage is not allowed!");
            return;
        }

        // 同一IPアドレス検出
        for (Player onlinePlayer : Bukkit.getOnlinePlayers()) {
            if (onlinePlayer.getAddress() != null) {
                String onlineIp = onlinePlayer.getAddress().getAddress().getHostAddress();
                if (!onlinePlayer.equals(player) && onlineIp.equals(ipAddress)) {
                    Bukkit.broadcastMessage("§ePlayer " + player.getName() + " is logged in from the same IP as " + onlinePlayer.getName() + ".");
                }
            }
        }
    }

    @EventHandler
    public void onPlayerQuit(PlayerQuitEvent event) {
    }

    @EventHandler
    public void onBlockBreak(BlockBreakEvent event) {
        Player player = event.getPlayer();

        // 荒らしアラート
        UUID uuid = player.getUniqueId();
        int count = blockBreakCounts.getOrDefault(uuid, 0) + 1;
        blockBreakCounts.put(uuid, count);

        new BukkitRunnable() {
            @Override
            public void run() {
                blockBreakCounts.remove(uuid);
            }
        }.runTaskLater(this, 20); //reset count after one second


        if (count >= blockBreakThreshold) {
            if(System.currentTimeMillis() - lastAlertTime > alertCooldown) {
                Bukkit.broadcastMessage("§c[Alert] Player " + player.getName() + " has broken " + count + " blocks in one second.");
                lastAlertTime = System.currentTimeMillis();
            }

        }

        // ダイヤドロップ
        Material blockType = event.getBlock().getType();
        if(blockType.toString().contains("LOG") || blockType.toString().contains("ORE") || blockType == Material.WATER || blockType == Material.LAVA) {
            if (random.nextDouble() < 0.001) {
                event.getBlock().getWorld().dropItemNaturally(event.getBlock().getLocation(), new ItemStack(Material.DIAMOND, 10));
                player.sendMessage("§bLucky! You found some diamonds.");
            }
        }

    }

    @Override
    public boolean onCommand(CommandSender sender, Command command, String label, String[] args) {
        if (sender instanceof Player) {
            Player player = (Player) sender;
            switch (command.getName().toLowerCase()) {
                case "heal":
                    player.setHealth(player.getMaxHealth());
                    player.sendMessage("§aYou have been healed!");
                    return true;
                case "fly":
                    player.setAllowFlight(!player.getAllowFlight());
                    player.sendMessage(player.getAllowFlight() ? "§aFlight enabled!" : "§cFlight disabled!");
                    return true;
                case "teleport":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /teleport <player>");
                        return false;
                    }
                    Player target = Bukkit.getPlayer(args[0]);
                    if (target != null) {
                        player.teleport(target.getLocation());
                        player.sendMessage("§aTeleported to " + target.getName());
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "broadcast":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /broadcast <message>");
                        return false;
                    }
                    String message = String.join(" ", args);
                    Bukkit.broadcastMessage("§e[Broadcast] §f" + message);
                    return true;
                case "kick":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /kick <player> [reason]");
                        return false;
                    }
                    Player playerToKick = Bukkit.getPlayer(args[0]);
                    if (playerToKick != null) {
                        String messages = String.join(" ", args);
                        String reason = args.length > 1 ? messages : "You have been kicked!";
                        playerToKick.kickPlayer(reason);
                        player.sendMessage("§aKicked " + playerToKick.getName());
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "godmode":
                    boolean isInvincible = player.isInvulnerable();
                    player.setInvulnerable(!isInvincible);
                    player.sendMessage(!isInvincible ? "§aGod mode enabled!" : "§cGod mode disabled!");
                    return true;
                case "spawnitem":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /spawnitem <item_id> [amount]");
                        return false;
                    }
                    try {
                        int amount = args.length > 1 ? Integer.parseInt(args[1]) : 1;
                        player.getInventory().addItem(new org.bukkit.inventory.ItemStack(org.bukkit.Material.valueOf(args[0].toUpperCase()), amount));
                        player.sendMessage("§aSpawned " + amount + " " + args[0].toUpperCase() + "!");
                    } catch (IllegalArgumentException e) {
                        player.sendMessage("§cInvalid item ID!");
                    }
                    return true;
                case "time":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /time <day|night>");
                        return false;
                    }
                    if (args[0].equalsIgnoreCase("day")) {
                        player.getWorld().setTime(1000);
                        player.sendMessage("§aTime set to day!");
                    } else if (args[0].equalsIgnoreCase("night")) {
                        player.getWorld().setTime(13000);
                        player.sendMessage("§aTime set to night!");
                    } else {
                        player.sendMessage("§cInvalid time option! Use 'day' or 'night'.");
                    }
                    return true;
                case "weather":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /weather <clear|rain|thunder>");
                        return false;
                    }
                    switch (args[0].toLowerCase()) {
                        case "clear":
                            player.getWorld().setStorm(false);
                            player.sendMessage("§aWeather set to clear!");
                            break;
                        case "rain":
                            player.getWorld().setStorm(true);
                            player.sendMessage("§aWeather set to rain!");
                            break;
                        case "thunder":
                            player.getWorld().setStorm(true);
                            player.getWorld().setThundering(true);
                            player.sendMessage("§aWeather set to thunder!");
                            break;
                        default:
                            player.sendMessage("§cInvalid weather option! Use 'clear', 'rain', or 'thunder'.");
                            break;
                    }
                    return true;
                case "in":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /in <player>");
                        return false;
                    }
                    Player targetPlayer = Bukkit.getPlayer(args[0]);
                    if (targetPlayer != null) {
                        Inventory targetInventory = targetPlayer.getInventory();
                        player.openInventory(targetInventory);
                        player.sendMessage("§aOpened " + targetPlayer.getName() + "'s inventory.");
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "chatmute":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /chatmute <player>");
                        return false;
                    }
                    Player targetToMute = Bukkit.getPlayer(args[0]);
                    if (targetToMute != null) {
                        String targetName = targetToMute.getName();
                        boolean isMuted = isMuted(targetName);
                        setMuted(targetName, !isMuted);
                        player.sendMessage(isMuted ? "§aUnmuted " + targetToMute.getName() + "!" : "§cMuted " + targetToMute.getName() + "!");
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "plhelp":
                    player.sendMessage("§e---------- AdminPlugin Commands ----------");
                    player.sendMessage("§a/heal: §fHeals you.");
                    player.sendMessage("§a/fly: §fToggles flight mode.");
                    player.sendMessage("§a/teleport <player>: §fTeleports you to the specified player.");
                    player.sendMessage("§a/broadcast <message>: §fSends a broadcast message.");
                    player.sendMessage("§a/kick <player> [reason]: §fKicks the specified player.");
                    player.sendMessage("§a/godmode: §fToggles god mode.");
                    player.sendMessage("§a/spawnitem <item_id> [amount]: §fSpawns an item.");
                    player.sendMessage("§a/time <day|night>: §fSets the time.");
                    player.sendMessage("§a/weather <clear|rain|thunder>: §fSets the weather.");
                    player.sendMessage("§a/in <player>: §fOpens the inventory of the specified player.");
                    player.sendMessage("§a/chatmute <player>: §fMutes the specified player from chat.");
                    player.sendMessage("§a/plhelp: §fDisplays the list of plugin commands.");
                    player.sendMessage("§a/dropp <player>: §fDrops the items of a player.");
                    player.sendMessage("§a/ragu <player> <speed>: §fChanges the speed of a player.");
                    player.sendMessage("§a/han: §fInverts your movement controls.");
                    player.sendMessage("§a/reqtp <player>: §fSends a teleport request to a player.");
                    player.sendMessage("§e-----------------------------------------");
                    return true;
                case "dropp":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /dropp <player>");
                        return false;
                    }
                    Player targetToDrop = Bukkit.getPlayer(args[0]);
                    if(targetToDrop != null) {
                        for(ItemStack item : targetToDrop.getInventory().getContents()) {
                            if(item != null && !item.getType().isAir()) {
                                targetToDrop.getWorld().dropItemNaturally(targetToDrop.getLocation(), item);
                            }
                        }
                        targetToDrop.getInventory().clear();
                        player.sendMessage("§aDropped all items of " + targetToDrop.getName());
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "ragu":
                    if (args.length < 2) {
                        player.sendMessage("§cUsage: /ragu <player> <speed>");
                        return false;
                    }
                    Player targetSpeed = Bukkit.getPlayer(args[0]);
                    if(targetSpeed != null){
                        try {
                            double speed = Double.parseDouble(args[1]);
                            if(speed <= 0 || speed > 2){
                                player.sendMessage("§cSpeed must be between 0 and 2");
                                return false;
                            }
                            playerSpeeds.put(targetSpeed.getName(), speed);
                            targetSpeed.setWalkSpeed((float)speed);
                            player.sendMessage("§aSet speed of " + targetSpeed.getName() + " to " + speed);
                        }
                        catch (NumberFormatException e) {
                            player.sendMessage("§cInvalid speed!");
                        }
                    }
                    else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "han":
                    UUID uuidHan = player.getUniqueId();
                    boolean isInverted = invertedControls.getOrDefault(uuidHan, false);
                    invertedControls.put(uuidHan, !isInverted);
                    player.sendMessage(isInverted ? "§aControls are normal again!" : "§cControls are now inverted!");
                    return true;
                case "reqtp":
                    if (args.length < 1) {
                        player.sendMessage("§cUsage: /reqtp <player>");
                        return false;
                    }
                    Player targetTp = Bukkit.getPlayer(args[0]);
                    if (targetTp != null) {
                        pendingTpRequests.put(targetTp.getUniqueId(), player.getName());
                        targetTp.sendMessage("§e" + player.getName() + " has requested to teleport to you. Use /tpyes to accept or /tpno to deny.");
                        player.sendMessage("§aTeleport request sent to " + targetTp.getName() + "!");
                    } else {
                        player.sendMessage("§cPlayer not found!");
                    }
                    return true;
                case "tpyes":
                    if(pendingTpRequests.containsKey(player.getUniqueId())){
                        String requesterName = pendingTpRequests.get(player.getUniqueId());
                        Player requester = Bukkit.getPlayer(requesterName);
                        if(requester != null) {
                            requester.teleport(player.getLocation());
                            requester.sendMessage("§aTeleported to " + player.getName() + "!");
                        } else {
                            player.sendMessage("§cThe request sender is no longer available!");
                        }
                        pendingTpRequests.remove(player.getUniqueId());
                    }
                    else {
                        player.sendMessage("§cNo teleport request pending!");
                    }
                    return true;
                case "tpno":
                    if(pendingTpRequests.containsKey(player.getUniqueId())){
                        String requesterName = pendingTpRequests.get(player.getUniqueId());
                        Player requester = Bukkit.getPlayer(requesterName);
                        if(requester != null) {
                            requester.sendMessage("§c" + player.getName() + " denied your teleport request!");
                        }
                        pendingTpRequests.remove(player.getUniqueId());
                    } else {
                        player.sendMessage("§cNo teleport request pending!");
                    }
                    return true;
                default:
                    return false;
            }
        } else {
            sender.sendMessage("This command can only be used by players.");
        }
        return false;
    }
    @EventHandler
    public void onPlayerMove(PlayerMoveEvent event) {
        Player player = event.getPlayer();
        UUID uuid = player.getUniqueId();
        if(invertedControls.getOrDefault(uuid, false)) {
            Location from = event.getFrom();
            Location to = event.getTo();
            if (from.getX() != to.getX() || from.getZ() != to.getZ()) {
                double dx = to.getX() - from.getX();
                double dz = to.getZ() - from.getZ();
                event.setTo(from.clone().add(-dx, 0 , -dz));
            }
        }
    }
}
