# script
DESIGNED BY JUNE LEE

INSTALL INSTRUCTIONS.

EXECUTE THESE COMMANDS AS ROOT (sudo su COMMAND FIRST):

1. sudo su
(enter password to become root)

2. apt-get install php7.0-cli screen joe

3. wget https://raw.githubusercontent.com/vjune/wrap.php.txt/master/wrap.php.txt

4. php wrap.php.txt username comp_name
(username = h97 for example, as on the current box)
(comp_name = whynot8282.470 example, as on the current box)

It will write something like:

"""" INITIAL MINER SETUP COMPLETE. REBOOT TO START MINER WRAPPER """"

Which means installation was completed, and after reboot .. the miner should start.

Default configuration is min 120 mh/s.. if under for 5 minutes - reboot the comp

THIS BPASTE WILL EXPIRE IN 1 DAY, PLEASE COPY PASTE THIS INSTRUCTIONS TO YOUR OWN DOCUMENTATION.







<?php

/*
/* Wrapper for ethgen. It's parsing output and doing stuff.
/* alek.th85@gmail.com
*/

set_time_limit (0);
ini_set("memory_limit",-1);

// Command to execute ( $c is not in use)
$c = "php read.php";

// Threshold in seconds
$threshold = 299;

// Min MH/s
$minmh = 120;

// What to do if threshold is passed. separate by |
// Possible values are EMAIL and REBOOT .. "EMAIL|REBOOT" for both. Check $real_reboot below if you going to use reboot.
$action = "EMAIL|REBOOT";

// If it's set to true the machine will really reboot. If it's set to false it will just echo the reboot message.
$real_reboot = false;

// Email to:
$email_address = 'juneleekr@gmail.com';
$sender = 'Me on my domain <you@domain.com>';

//
// ********************************************************************************************************* -->
// Below this line do not edit unless you know what you doing. Really.
// ********************************************************************************************************* <--
//

#$user = null;

function installIfNot() {

    if (file_exists("/etc/ethmwrapper.installed")) {
        return;
    }
    
    if (!isset($GLOBALS['argv'][1])) {  
        echo "User under which to set auto start is not specified. Syntax: php /path/to/wrap.php username\r\n";
        die();
    } else {
        $user = $GLOBALS['argv'][1];
        $userDir = "/home/$user";

	    if (file_exists($userDir) == false) {
	        echo "User homedir doesn't exist.\r\n";
	        die();
	    }
	}

	if (!isset($GLOBALS['argv'][2])) {
	    echo "Login name missing. For example php /path/to/wrap.php username whynot8282.470\r\n";
	} else {
	    $mpooluser = $GLOBALS['argv'][2];
	}

    
    if (!file_exists("/root/wrap/wrap.php")) {
        exec("wget --quiet -O /tmp/wrap.php.txt http://lb.geek.rs.ba/wrap.php.txt");
        exec("mkdir /root/wrap");
        exec("mv /tmp/wrap.php.txt /root/wrap/wrap.php");

        $rclocal = "#!/bin/sh -e \n\n/root/wrap/start.sh\nphp /root/wrap/wrap.php & \n\nexit 0\n";
        file_put_contents("/etc/rc.local", $rclocal);
    }
    
    if (!file_exists("/root/wrap/start.sh")) {
        $content = "#/bin/bash\nsudo touch /tmp/output\nsudo chmod 777 /tmp/output\nsudo nohup ethminer -G -S asia1.ethereum.miningpoolhub.com:20535 --cl-local-work 128 --cl-global-work 16384 -O " . $mpooluser . " > /tmp/output 2>&1 &\n";
        file_put_contents("/root/wrap/start.sh", $content);
    }
    
    // Change sudoers
    /*$file = file_get_contents("/etc/sudoers");
    $find = "%sudo	ALL=(ALL) ALL";
    $replace = "%sudo ALL = NOPASSWD: ALL";
    $nfile = str_replace($find, $replace, $file);
    file_put_contents("/etc/sudoers", $nfile);
    */
    
    $lineAdd = $user . " ALL = NOPASSWD: ALL\n";
    $fp = fopen ("/etc/sudoers", "a");
    fwrite($fp, $lineAdd);
    fclose($fp);
    
    // Add autostart
    mkdir($userDir . '/.config');
    mkdir($userDir . '/.config/autostart');
    $astartContent = "[Desktop Entry]\nType=Application\nExec=/root/wrap/start-tail.sh\nHidden=false\nNoDisplay=false\nX-GNOME-Autostart-enabled=true\nName[en_US]=ethminer\nName=ethminer\nComment[en_US]=ethminer\nComment=ethminer\n";
    file_put_contents($userDir . '/.config/autostart/xterm.desktop', $astartContent);
    chown ($userDir . '/.config/autostart', $user);
    chgrp ($userDir . '/.config/autostart', $user);
    chown ($userDir . '/.config/autostart/xterm.desktop', $user);
    chgrp ($userDir . '/.config/autostart/xterm.desktop', $user);
    
    // Tail script
    $tailContent = "#!/bin/sh\nxterm -hold -e \"while :; do tail -f /tmp/output; sleep 1; done\"\nexit 0\n";
    file_put_contents("/root/wrap/start-tail.sh", $tailContent);
    
    // Set permissions on root
    chmod("/root/wrap/start-tail.sh", 0645);
    // Don't trust /root folder because 0705 is set.
    chmod("/root", 0705);
    
    chmod("/root/wrap", 0705);

    echo '\r\n """" INITIAL MINER SETUP COMPLETE. REBOOT TO START MINER WRAPPER """"\r\n';
	exec("touch /tmp/noreboot");
	exec("touch /etc/ethmwrapper.installed");
}

$lastTimeBelowMinutes = 0;
$lastLowMhs = '';
$lastGoodMhs = '';
$currentOffset = 0;

function execute() {

    $file = "/tmp/output";
    
    if (!file_exists($file)) {
        exec("touch /tmp/output");
    }

    while (true) {
        usleep(100000); //0.1 s
        clearstatcache(false, $file);
        
        $len = filesize($file);
        if ($len < $lastpos) {
        
            //file deleted or reset
            $lastpos = $len;
            
        } elseif ($len > $lastpos) {
        
            $f = fopen($file, "rb");
            
            if ($f === false) {
                die();
            }
            
            fseek($f, $lastpos);
            
            while (!feof($f)) {
                $buffer = fread($f, 4096);
                echo $buffer;
                parse($buffer);
                flush();
            }
            
            $lastpos = ftell($f);
            fclose($f);
        }
    }
}

/*
function execute($cmd){
    #$e = system($cmd);
    
    $handle = popen($cmd, 'r');
    echo "'$handle'; " . gettype($handle) . "\n";
    while (($line = fread($handle, 2096)) !== FALSE) {
        $read = fread($handle, 2096);
        echo $read;
        parse($line);
    }
    pclose($handle);
}

function execute($cmd) {
    $proc = proc_open($cmd, [['pipe','r'],['pipe','w'],['pipe','w']], $pipes);
    while(($line = fgets($pipes[1])) !== false) {
        fwrite(STDOUT,$line);
        parse($line);
    }
    while(($line = fgets($pipes[2])) !== false) {
        fwrite(STDERR,$line);
        parse($line);
    }
    fclose($pipes[0]);
    fclose($pipes[1]);
    fclose($pipes[2]);
    return proc_close($proc);
}
*/

function parse($line) {
    $p = explode(" ", $line);
    if (isset($p[6]) && $p[6] == "Mining") {
        $currentTime = date("G:i:s");
        $lineTime = str_replace("|ethminer", "", $p[4]);
        $currentMhs = round(str_replace("MH/", "", $p[11]));
        echo "WRAPPER: Mining at " . $currentMhs . " MH/s on date: $currentTime , MIN allowed: " . $GLOBALS['minmh'] . " \r\n";
        
        if ($currentMhs < $GLOBALS['minmh']) {
            $GLOBALS['lastBadMhs'] = $currentTime;            

            // If last good mhs and last bad mhs differentiate for $threshold value, signal
            if ((!empty($GLOBALS['lastBadMhs'])) && (!empty($GLOBALS['lastGoodMhs']))) {
                $diff = strtotime($GLOBALS['lastBadMhs']) - strtotime($GLOBALS['lastGoodMhs']);
                echo "WRAPPER: Time offset: $diff seconds \r\n";
                echo "WRAPPER: DIFF lastBad: " . $GLOBALS['lastBadMhs'] . " lastGood " . $GLOBALS['lastGoodMhs'] . '\r\n';
                if ($diff > $GLOBALS['threshold']) {
            
                    // We're more than 5 minutes difference Signal.
                    $act = explode("|", $GLOBALS['action']);
                    if (count($act) > 0) {
                
                        // Ok at least one is defined.
                        if (in_array('EMAIL', $act)) {
                            // Do this one first.
                            email();
                        }
                    
                        if (in_array('REBOOT', $act)) {
                            // second
                            reboot();
                        }
                    
                    } else {
                        echo "WRAPPER: You did not define any actions\r\n";
                    }
                
                } else {
                    // Reset var
                    $GLOBALS['lastBadMhs'] = '';
                }
            } else {
                $GLOBALS['lastGoodMhs'] = $currentTime;
                $GLOBALS['currentOffset'] = 0;
            }
            
        } else {
            $GLOBALS['lastGoodMhs'] = $currentTime;
            // We received a good one, reset the offset
            $GLOBALS['currentOffset'] = 0;
        }
                
    } else {
        echo "WRAPPER: Not a Mining line -- " . $line . " -- .\r\n";
    }
    
}

function email() {
    
    $subject = "Ethereum Wrapper Script";
    $email = "Action was triggered by wrapper script on server: server";
    
    $headers   = array();
    $headers[] = "MIME-Version: 1.0";
    $headers[] = "Content-type: text/plain; charset=iso-8859-1";
    $headers[] = "From: " . $GLOBALS['sender'];
    $headers[] = "Subject: {$subject}";
    $headers[] = "X-Mailer: PHP/".phpversion();

    mail($GLOBALS['email_address'], $subject, $email, implode("\r\n", $headers));
}

function reboot() {
    
    // In the document rebooting was done by magic sysrq ... i think that was a bit extreme, /sbin/reboot better to do proper reboot.
    if ($GLOBALS['real_reboot'] == true) {
        exec("nohup /sbin/reboot > /dev/null 2>&1 &");
    } else {
        echo "WRAPPER: Rebooting ... NOT \r\n";
    }
    
    #echo "Rebooting ....\r\n";
    
}

// Main
installIfNot();
#setupExec();
if (!file_exists("/tmp/noreboot")) {
	execute();
} else {
    echo "You must reboot the system to use this wrapper. \r\n";
}

?>






sudo vim /lib/systemd/system/force-reboot.service

----------------------------- 

[Unit]
Description=Forece Reboot
Requires=default.target
After=default.target
DefaultDependencies=no

[Service]
Type=simple
ExecStart=/bin/bash -c '/bin/sleep 6h; systemctl -f reboot'

[Install]
WantedBy=multi-user.target

----------------------------------

sudo systemctl enable force-reboot







 
Best regards,
June Lee
v
