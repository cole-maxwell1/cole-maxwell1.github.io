---
date: 2022-08-03 19:39:00 -05:00
title: Using ODBC on IBM i for Local Linux Development
categories: [IBM i,]
tags: [ibmi, odbc, linux, as400, sql]
---

If you are new to the IBM i platform coming right out of school, like me, or you are a developer used to working exclusively with open-source tooling, the IBM i platform can be a strange place. The legacy application support is industry leading, to both the benefit and downside of the platform. Under the hood there is IBM's powerful and well tested relational database, DB2 for i. So far, I like what I see. Is it possible to get the best of the open-source tooling while leveraging the power of an existing enterprize grade database? My initial take is yes, and the ODBC driver on the platform is the answer.

The [Open Database Connectivity](https://en.wikipedia.org/wiki/Open_Database_Connectivity) (ODBC) standard is an application programming interface (API) for accessing database management systems (DBMS). The designers of ODBC aimed to make it independent of database systems and operating systems. From what I have seen so far IBM is continuing to make improvements to the driver in the latest release of IBM i. I personally view ODBC as the future of the platform and a key factor to help IBM i remain relevant as the years progress.

As I have gotten my start, the other open-source advocates on the IBM i platform have provided many excellent articles, source code examples, and video lessons that have aided my on boarding to the platform. However, even though the topic of ODBC has been extensively covered I still felt like a comprehensive guide for connecting locally to DB2 for i via ODBC on a linux development machine was missing.

This post has compiled the good, but scattered, information to explain how an open-source developer would go about connecting up their local linux machine to DB2 for i to develop an application in an open-source language of their choice. I want to give a big thank you and credit to **Liam Allan**, **Kevin Adler**, **Seiden Group**, and **FormaServe** for helping get this information out to the community. They are the heavy hitters aiding the open-source embracement on IBM i. Much of the following is copied directly from their blogs and video resources. Please see the [references](#references) at the end of this post.

# Prerequisites
You must have the ODBC driver installed on the IBM i you want to connect to. See Seiden Group's guide[^Seiden] for [Using YUM to Install or Update the IBM i ODBC Driver](https://www.seidengroup.com/2022/07/11/using-yum-to-install-or-update-the-ibm-i-odbc-driver/)

### Enhance Your Understanding
If your background is exclusively developing on the IBM i (AS400) and you don't have experience with Linux or other Unix-like operating systems you may want to spend some time reading about the basic file structure of Unix-like operating systems. You will find these patterns in the `/QOpenSys` directory on IBM i and it will aid your journey to better understand open-source on IBM i and even Linux.

* University of Cincinnati: [The Unix File System](https://homepages.uc.edu/~thomam/Intro_Unix_Text/File_System.html)
* IBM: [Open systems file system (QOpenSys)](https://www.ibm.com/docs/en/i/7.5?topic=systems-open-file-system-qopensys)

# Installing the Repository
IBM has made RPM and DEB package manager repositories for Linux available directly from IBM for the `IBM i Access Client Solutions` application package, which includes the `IBM i Access ODBC driver`.

With this change, it is much easier to install the driver on Linux. It also makes it easier for automation to install the driver as well, whether that’s Ansible system deployment scripts or Dockerfiles for building ODBC-based Linux container apps. In addition, it makes updating the driver much easier too, since the process uses the same upgrade procedure as the rest of the system packages[^Adler].

The repositories are located under: [https://public.dhe.ibm.com/software/ibmi/products/odbc/](https://public.dhe.ibm.com/software/ibmi/products/odbc/).

## Add the Repository to the Package Manager

First, you must add IBM's package repository to your distribution's package manager.

### Debian-based and Ubuntu-based Distribution Setup
``` shell
curl https://public.dhe.ibm.com/software/ibmi/products/odbc/debs/dists/1.1.0/ibmi-acs-1.1.0.list | sudo tee /etc/apt/sources.list.d/ibmi-acs-1.1.0.list
```
### Red Hat-based Distribution Setup
``` shell
curl https://public.dhe.ibm.com/software/ibmi/products/odbc/rpms/ibmi-acs.repo | sudo tee /etc/yum.repos.d/ibmi-acs.repo
```
### SUSE-based Distribution Setup
``` shell
curl https://public.dhe.ibm.com/software/ibmi/products/odbc/rpms/ibmi-acs.repo | sudo tee /etc/zypp/repos.d/ibmi-acs.repo
```

# Installing the ODBC driver
Now install the package via the distribution's package manager.
### Debian-based and Ubuntu-based Distribution Installation
``` shell
sudo apt update
sudo apt install ibm-iaccess
```
### Red Hat-based Distribution Installation
``` shell
sudo dnf install --refresh ibm-iaccess
```
### SUSE-based Distribution Installation
``` shell
sudo zypper refresh
sudo zypper install ibm-iaccess
```

# Configuring the Connection (FYI)

Now that you have the IBM i Access ODBC Driver installed on your system, you are ready to connect to Db2 on i. Below is some background information on the two methods to create a connection.

## Connection Strings
ODBC uses a connection string with keywords to create a database connection. Keywords are case insensitive, and values passed are separated from the keyword by an equals sign (“=”) and end with a semi-colon (“;”). As long as you are using an ODBC database connector, you should be able to pass an identical connection string in any language or technology and be confident that it will correctly connect to Db2 on i. A common connection string may look something like[^odbc-docs]:

```
DRIVER=IBM i Access ODBC Driver;SYSTEM=my.ibmi.system;UID=foo;PWD=bar;
```
In the above example, we define the following connection options:

* DRIVER: The ODBC driver for Db2 for i that we are using to connect to the database (and that we installed above)
* SYSTEM: The location of your IBM i system, which can be its network name, IP address, or similar
* UID: The User ID that you want to use on the IBM i system that you are connecting to
* PWD: The password of the User ID passed above.

These are only some of the over 70 connection options you can use when connecting to Db2 on i using the IBM i Access ODBC Driver. A complete list of IBM i Access ODBC Driver connection options can be found at the [IBM Knowledge Center: Connection string keywords webpage](https://www.ibm.com/docs/en/i/7.4?topic=details-connection-string-keywords). If passing connections options through the connection string, be sure to use the keyword labeled with **Connection String**.

## DSNs
As you add more and more options to your connection string, your connection string can become quite cumbersome. Luckily, ODBC offers another way of defining connection options called a DSN (datasource name). Where you define your DSN will depend on whether you are using Windows ODBC driver manager or unixODBC on Linux or IBM i[^odbc-docs].

IBM i, Linux distributions, and macOS use unixODBC and have nearly identical methods of setting up your drivers and your DSNs.

`odbc.ini` and `.odbc.ini`

When using unixODBC, DSNs are defined in `odbc.ini` and `.odbc.ini` (note the `.` preceding the latter). These two files have the same structure, but have one important difference:

`odbc.ini` defines DSNs that are available to all users on the system. If there are DSNs that should be available to everyone, they can be defined and shared here. Likely, this file is located in the default location, which depends on whether you are on IBM i or Linux:

IBM i: `/QOpenSys/etc/odbc.ini`

Linux: `/etc/unixODBC/odbc.ini`

If you want to make sure, the file can be found by running:
```
odbcinst -j
```
.odbc.ini is found in your home directory `~/` and defines DSNs that are available only to you. If you are going to define DSNs with your personal username and password, this is the place to do it.

In both `odbc.ini` and `.odbc.ini`, you name your DSN with `[]` brackets, then specify keywords and values below it. An example of a DSN stored in `~/.odbc.ini` used to connect to an IBM i system with private credentials might look like:

```
[MYDSN]
Description            = My IBM i System
Driver                 = IBM i Access ODBC Driver
System                 = my.ibmi.system
UserID                 = foo
Password               = bar
Naming                 = 0
DefaultLibraries       = MYLIB
TrueAutoCommit         = 1
```

In the above example, we define the following connection options:

* Driver: The ODBC driver for Db2 for i that we are using to connect to the database (and that we installed above)
* System: The location of your IBM i system, which can be its network name, IP address, or similar
* UserID: The User ID that you want to use on the IBM i system that you are connecting to
* Password: The password of the User ID passed above.
* Naming: Specifies the naming convention used when referring to tables. For more information, refer to Naming [conventions](https://www.ibm.com/docs/en/ssw_ibm_i_74/db2/rbafzch2nam.htm) in the DB2 for i SQL reference.
* DefaultLibraries: Specifies the IBM i libraries to add to the server job's library list as well as the default library used to resolve unqualified names. The libraries can be delimited by commas or spaces.
* TrueAutoCommit: Specifies how to handle autocommit support

Like connection string keywords, DSN keywords can be found at the [IBM Knowledge Center: Connection string keywords webpage](https://www.ibm.com/support/knowledgecenter/ssw_ibm_i_74/rzaik/connectkeywords.htm). When passing connection options through a DSN, be sure to use the keyword labeled with ODBC.INI.

# User Level Connection

1. In your home directory create a `.odbc.ini` file

```
touch .odbc.ini
```
2. Using your text editor of choice ([vi](https://www.redhat.com/sysadmin/introduction-vi-editor), [vim](https://linuxfoundation.org/blog/classic-sysadmin-vim-101-a-beginners-guide-to-vim/), [nano](https://www.howtogeek.com/howto/42980/the-beginners-guide-to-nano-the-linux-command-line-text-editor/), ect...) add the following DSN configuration, changing the `UserID` and `Password` to **your** IBM i username and password.

```
[devserver]
Description            = Connection to development power server
Driver                 = IBM i Access ODBC Driver
System                 = devserver.mycompany.com
UserID                 = myUsername
Password               = myPassword
Naming                 = 0
DefaultLibraries       = businessDataLib,myUserLib,myBusinessUtils
```

In the above example, we define the following connection options:
* Driver: The ODBC driver for Db2 for i that we are using to connect to the database (This must be installed on the OS)
* System: The location of your IBM i system, which can be its network name, IP address, or similar
* UserID: The User ID that you want to use on the IBM i system that you are connecting to
* Password: The password of the User ID passed above.
* Naming: Specifies the naming convention used when referring to tables. A `0` indicates you want SQLs naming convention, which is likely what you want for running queries in any program you write.
* DefaultLibraries: Specifies the IBM i libraries to add to the server job's library list as well as the default library used to resolve unqualified names. The libraries can be delimited by commas or spaces.

# Testing the Connection

The `isql` command is included with the open source components of unixODBC and unixODBC-devel. You can use this command to run SQL queries via ODBC right from the command-line. To make the connection to the **DSN** configured above pass in the DSN name:

```
isql devserver
```

If the connection is configured correctly, you will get a prompt that looks like this:

```
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
SQL>
```

There is an included sample customer table (`qiws.qcustcdt`) on all IBM i systems that can be used for testing[^FormaServe]

Type this command into the `SQL>` prompt:
```
select * from qiws.qcustcdt
```

The results will look like this:
```
SQL> select * from qiws.qcustcdt
+---------+---------+-----+--------------+-------+------+--------+-------+-------+---------+---------+
| CUSNUM  | LSTNAM  | INIT| STREET       | CITY  | STATE| ZIPCOD | CDTLMT| CHGCOD| BALDUE  | CDTDUE  |
+---------+---------+-----+--------------+-------+------+--------+-------+-------+---------+---------+
| 938472  | Henning | G K | 4859 Elm Ave | Dallas| TX   | 75217  | 5000  | 3     | 37.00   | 0       |
| 839283  | Jones   | B D | 21B NW 135 St| Clay  | NY   | 13041  | 400   | 1     | 100.00  | 0       |
| 392859  | Vine    | S S | PO Box 79    | Broton| VT   | 5046   | 700   | 1     | 439.00  | 0       |
| 938485  | Johnson | J A | 3 Alpine Way | Helen | GA   | 30545  | 9999  | 2     | 3987.50 | 33.50   |
| 397267  | Tyron   | W E | 13 Myrtle Dr | Hector| NY   | 14841  | 1000  | 1     | 0       | 0       |
| 389572  | Stevens | K L | 208 Snow Pass| Denver| CO   | 80226  | 400   | 1     | 58.75   | 1.50    |
| 846283  | Alison  | J S | 787 Lake Dr  | Isle  | MN   | 56342  | 5000  | 3     | 10.00   | 0       |
| 475938  | Doe     | J W | 59 Archer Rd | Sutter| CA   | 95685  | 700   | 2     | 250.00  | 100.00  |
| 693829  | Thomas  | A N | 3 Dove Circle| Casper| WY   | 82609  | 9999  | 2     | 0       | 0       |
| 593029  | Williams| E D | 485 SE 2 Ave | Dallas| TX   | 75218  | 200   | 1     | 25.00   | 0       |
| 192837  | Lee     | F L | 5963 Oak St  | Hector| NY   | 14841  | 700   | 2     | 489.50  | .50     |
| 583990  | Abraham | M T | 392 Mill St  | Isle  | MN   | 56342  | 9999  | 3     | 500.00  | 0       |
+---------+---------+-----+--------------+-------+------+--------+-------+-------+---------+---------+
SQLRowCount returns -1
12 rows fetched
SQL>
```

To exit the prompt press <kbd>Ctr + C</kbd> or type `quit` into the prompt

# Next Steps

Now that you have the ODBC driver installed and configured on you development machine you can use the ODBC drivers in many of your favorite open-source programing languages. See IBM's [Db2 for Developers](https://www.ibm.com/products/db2-database/developers?utm_content=SRCWW&p1=Search&p4=43700068101138376&p5=p&gclid=Cj0KCQjwuaiXBhCCARIsAKZLt3npUFCKyE-dxmuikHbggZepKbmypy53W5utWVdqpc5xsTI1Jqk_FHQaAusYEALw_wcB&gclsrc=aw.ds) page for supported languages.

Examples of connections in programing languages:
* Seiden Group: [HOW TO QUERY IBM i DATA WITH PHP AND PDO_ODBC](https://www.seidengroup.com/2022/04/15/how-to-query-ibm-i-data-with-php-and-pdo_odbc/)
* Liam Allan: [Using node-odbc on Windows to talk to IBM i](https://worksofbarry.com/?post=25)

# Other Helpful Articles:
* Seiden Group: [Q&A: IBM i ODBC DRIVER](https://www.seidengroup.com/2020/08/28/qa-ibm-i-odbc-driver/)
* Seiden Group: [ODBC CONNECTION STRINGS FOR IBM i DB2](https://www.seidengroup.com/2022/05/18/odbc-connection-strings-for-ibm-i-db2/)

# References
[^Adler]: Kevin Adler: [IBM-provided Repositories ODBC Linux Driver Repositories](https://kadler.io/2022/05/20/odbc-repos.html)
[^FormaServe]: FormaServe: [How to use ODBC on the IBM i](https://www.youtube.com/watch?v=hOFZf4bd_wM)
[^Allan]: Liam Allan: [Deploying a Node.js + Db2 for i app using Docker (from a Codespace!)](https://worksofbarry.com/?post=59)
[^odbc-docs]: [IBM i OSS Docs](https://ibmi-oss-docs.readthedocs.io/en/latest/odbc/using.html)
[^Seiden]: Seiden Group: [USING YUM TO INSTALL OR UPDATE THE IBM i ODBC DRIVER](https://www.seidengroup.com/2022/07/11/using-yum-to-install-or-update-the-ibm-i-odbc-driver/)

  
