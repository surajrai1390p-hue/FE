# Experiment 9 -- Contact Form with Vercel, ngrok, PHP and MySQL

## Aim

To create a contact form whose frontend is deployed on Vercel and whose
PHP backend and MySQL database run on a Kali Linux virtual machine.
ngrok exposes the local Apache server so that the deployed frontend can
send form submissions to it.

## 1. Architecture

``` text
Browser
  |
  v
Frontend on Vercel (index.html)
  |
  | HTTP POST
  v
Public ngrok URL
  |
  | forwards to localhost:80
  v
Kali Linux: Apache + PHP (submit.php)
  |
  v
MySQL: web_project.contacts
```

**Important:** Vercel hosts only the frontend. The PHP backend and MySQL
database remain on Kali Linux. Kali, Apache, MySQL and ngrok must be
running when the form is used.

## 2. Requirements

-   Kali Linux running in VirtualBox
-   Internet connection
-   Apache, PHP and PHP-MySQL extension
-   MySQL server (or the MySQL-compatible database package supplied by
    the Kali repository)
-   ngrok account and authentication token
-   Windows computer with VS Code
-   GitHub account and Vercel account

------------------------------------------------------------------------

## 3. Optional: Reset only the previous Experiment 9 work

Do this only if you want to remove the old project and database before
practising again. These commands do **not** reset all Kali Linux
settings.

Remove the old project:

``` bash
sudo rm -rf /var/www/html/contact-form
```

Remove the database and experiment user:

``` bash
sudo mysql
```

Run inside the SQL prompt:

``` sql
DROP DATABASE IF EXISTS web_project;
DROP USER IF EXISTS 'phpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Remove the ngrok configuration:

``` bash
rm -rf ~/.config/ngrok
```

If ngrok was installed manually at `/usr/local/bin/ngrok`, remove that
binary too:

``` bash
sudo rm -f /usr/local/bin/ngrok
```

If you still need ngrok for another project, do not remove its
configuration or binary.

------------------------------------------------------------------------

## 4. Install Apache, PHP and MySQL

Open the Kali Linux terminal.

Update package information:

``` bash
sudo apt update
```

Install the required packages:

``` bash
sudo apt install apache2 php libapache2-mod-php mysql-server php-mysql -y
```

If Kali reports that `mysql-server` is unavailable, check the database
package available in your configured Kali repositories. Kali
installations commonly provide MariaDB as a MySQL-compatible server.
Install the available MariaDB server package instead, then use the
`mysql` client commands below.

Start the services:

``` bash
sudo service apache2 start
sudo service mysql start
```

Check PHP:

``` bash
php -v
```

Check Apache in the Kali browser:

``` text
http://localhost
```

The Apache default page should appear.

Check the database login:

``` bash
sudo mysql
```

If the SQL prompt opens, exit for now:

``` sql
EXIT;
```

------------------------------------------------------------------------

## 5. Create the database and table

Open the database prompt:

``` bash
sudo mysql
```

Create the database and table:

``` sql
CREATE DATABASE web_project;

USE web_project;

CREATE TABLE contacts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    message TEXT NOT NULL
);
```

Create a separate user for the PHP application. Replace
`CHANGE_THIS_PASSWORD` with a strong password of your choice and
remember it for the PHP file.

``` sql
CREATE USER 'phpuser'@'localhost'
IDENTIFIED BY 'CHANGE_THIS_PASSWORD';

GRANT INSERT, SELECT ON web_project.*
TO 'phpuser'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

The PHP connection will use this database username and password.

------------------------------------------------------------------------

## 6. Create the PHP backend on Kali Linux

Create the project directory inside Apache's document root:

``` bash
sudo mkdir -p /var/www/html/contact-form
sudo chown -R "$USER":"$USER" /var/www/html/contact-form
cd /var/www/html/contact-form
```

Create the backend file:

``` bash
nano submit.php
```

Paste the following code. Replace `CHANGE_THIS_PASSWORD` with the same
password used when creating `phpuser`.

``` php
<?php

$conn = new mysqli("localhost", "phpuser", "suraj", "web_project");

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$name = $_POST['name'];
$email = $_POST['email'];
$message = $_POST['message'];

$sql = "INSERT INTO contacts (name, email, message)
        VALUES ('$name', '$email', '$message')";

if ($conn->query($sql) === TRUE) {
    echo "Message submitted successfully!";
} else {
    echo "Error: " . $conn->error;
}

$conn->close();

?>

```

Save in nano: press `Ctrl + O`, press `Enter`, then `Ctrl + X`.

Check that PHP is accessible through Apache:

``` text
http://localhost/contact-form/submit.php
```

Opening this URL with a browser sends a GET request, so the expected
response is `Method not allowed`. This confirms that Apache is reaching
the PHP file; it does not yet test database insertion.

------------------------------------------------------------------------

## 7. Test the backend before connecting Vercel

On Kali Linux, make sure Apache and MySQL are running:

``` bash
sudo service apache2 start
sudo service mysql start
```

You can test the PHP endpoint from the Kali terminal using:

``` bash
curl -i -X POST http://localhost/contact-form/submit.php \
  -d "name=Test User" \
  -d "email=test@example.com" \
  --data-urlencode "message=Testing Experiment 9"
```

A successful response should say:

``` text
Contact form submitted successfully!
```

Check the database:

``` bash
sudo mysql
```

``` sql
USE web_project;
SELECT * FROM contacts;
EXIT;
```

Confirm that the test record appears. Do not continue until the local
backend test works.

------------------------------------------------------------------------

## 8. Create the frontend on Windows using VS Code

Create a folder named `contact-frontend` on Windows and open it in VS
Code.

Create a file named `index.html` in that folder.

Paste this code:

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Contact Form</title>
</head>
<body>
    <h1>Contact Us</h1>

    <form
        action="https://YOUR-NGROK-DOMAIN/contact-form/submit.php"
        method="POST"
    >
        <label for="name">Name:</label><br>
        <input type="text" id="name" name="name" required><br><br>

        <label for="email">Email:</label><br>
        <input type="email" id="email" name="email" required><br><br>

        <label for="message">Message:</label><br>
        <textarea id="message" name="message" required></textarea><br><br>

        <button type="submit">Submit</button>
    </form>
</body>
</html>
```

Keep the form field names exactly as `name`, `email`, and `message`,
because the PHP backend reads those names.

The ngrok address is not known yet. Leave the placeholder for now.
Replace it after starting ngrok in the next section.

------------------------------------------------------------------------

## 9. Install and configure ngrok on Kali Linux

Create a setup directory:

``` bash
mkdir -p ~/ngrok-setup
cd ~/ngrok-setup
```

Download the Linux ngrok archive from the official ngrok download page:

``` text
https://ngrok.com/download
```

Follow the current Linux installation instructions provided for your
Kali architecture. Extract the downloaded archive and install the
`ngrok` binary. If using the archive shown in the experiment journal,
the commands are:

``` bash
tar -xvzf ngrok-v3-stable-linux-amd64.tgz
sudo mv ngrok /usr/local/bin/
```

Check installation:

``` bash
ngrok version
```

Create an ngrok account and copy your authentication token from:

``` text
https://dashboard.ngrok.com/get-started/your-authtoken
```

Add the token:

``` bash
ngrok config add-authtoken YOUR_AUTH_TOKEN
```

Do not share your authentication token or publish it in GitHub.

------------------------------------------------------------------------

## 10. Start ngrok and obtain the public URL

Make sure Apache is running:

``` bash
sudo service apache2 start
```

Start the tunnel:

``` bash
ngrok http 80
```

Keep this terminal open. ngrok displays a forwarding address similar to:

``` text
https://example.ngrok-free.app
```

Copy the actual HTTPS forwarding address shown in your terminal.

The PHP endpoint will be:

``` text
https://YOUR-ACTUAL-NGROK-DOMAIN/contact-form/submit.php
```

Test that URL in a browser. A `Method not allowed` response is expected
for a GET request.

**Important:** The ngrok URL is temporary on many free ngrok setups and
may change when the tunnel restarts. If it changes, update the frontend
form action and redeploy the frontend.

------------------------------------------------------------------------

## 11. Connect the frontend to ngrok

Return to `index.html` in VS Code.

Replace:

``` html
action="https://YOUR-NGROK-DOMAIN/contact-form/submit.php"
```

with the actual URL shown by ngrok, for example:

``` html
action="https://example.ngrok-free.app/contact-form/submit.php"
```

Use your own current ngrok URL, not the example.

Save the file.

This version uses a normal HTML POST form. A normal cross-origin form
submission does not require JavaScript CORS configuration. The browser
will navigate to the PHP response after submission.

------------------------------------------------------------------------

## 12. Deploy the frontend on Vercel

1.  Open <https://vercel.com/>.
2.  Sign in or create an account.
3.  Upload the `contact-frontend` project to a GitHub repository, or
    import an existing repository.
4.  In Vercel, select **Add New → Project**.
5.  Import the frontend repository.
6.  For a plain HTML project, use the appropriate static deployment
    settings. No build command is required; the `index.html` file should
    be in the deployed project root.
7.  Click **Deploy**.
8.  Open the Vercel deployment URL.

Only the frontend folder should be deployed to Vercel. Do not upload
database passwords, ngrok authentication tokens, or private backend
configuration files to the frontend repository.

------------------------------------------------------------------------

## 13. Test the complete application

Before testing, confirm that all of these are running:

-   Kali Linux VM is powered on and connected to the internet.
-   Apache is running.
-   MySQL is running.
-   ngrok tunnel is running and shows an active forwarding URL.
-   The Vercel form's `action` contains the current ngrok URL.

Open the deployed Vercel website.

1.  Enter a name.
2.  Enter a valid email address.
3.  Enter a message.
4.  Click **Submit**.
5.  The PHP backend should display
    `Contact form submitted successfully!` if the record is saved.

Now verify the database on Kali:

``` bash
sudo mysql
```

``` sql
USE web_project;
SELECT * FROM contacts;
```

The submitted contact should appear in the table.

------------------------------------------------------------------------

## 14. Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  Apache page does not open           Run `sudo service apache2 start`

  PHP code is displayed as text or    Check Apache PHP module
  downloaded                          installation and configuration

  Database connection failed          Check MySQL status, database name,
                                      username and password

  `404 Not Found`                     Check the PHP file path and URL

  `405 Method not allowed` in browser Expected for GET; submit the form
                                      using POST

  ngrok URL does not open             Check that ngrok is running and
                                      Apache is listening on port 80

  Form submits to an old URL          Update the form action with the
                                      current ngrok URL

  Record is not inserted              Check PHP validation, database
                                      permissions and MySQL table

  Vercel page not found               Check that `index.html` is in the
                                      deployed project root
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 15. Important security notes

-   Use a strong MySQL password instead of the sample password.
-   Use prepared statements for SQL queries.
-   Never publish your ngrok authentication token or database password.
-   Do not expose MySQL port 3306 publicly. Only expose the Apache HTTP
    service through ngrok.
-   The PHP example is suitable for a learning experiment, not a
    production contact form. A production application should also
    implement CSRF protection, spam prevention, rate limiting, HTTPS,
    and secure secret management.
-   Anyone who can access the public ngrok endpoint may be able to
    submit the form. Stop the tunnel when you finish practising.

------------------------------------------------------------------------

## 16. Final verification checklist

-   [ ] Apache installed and running
-   [ ] PHP installed and working
-   [ ] MySQL installed and running
-   [ ] `web_project` database created
-   [ ] `contacts` table created
-   [ ] `phpuser` database account created
-   [ ] `submit.php` created and tested locally
-   [ ] Local test record visible in MySQL
-   [ ] Frontend `index.html` created
-   [ ] ngrok installed and authenticated
-   [ ] ngrok tunnel forwarding to port 80
-   [ ] Frontend form action updated with current ngrok URL
-   [ ] Frontend deployed on Vercel
-   [ ] Vercel form submission successful
-   [ ] Submitted record visible in MySQL

## Conclusion

The contact form frontend is deployed on Vercel, while the PHP backend
and MySQL database run on Kali Linux. ngrok provides a public URL that
forwards requests to the local Apache server. When a user submits the
contact form, PHP receives the data and stores it in the MySQL
`contacts` table.
