# Web Connection Security

<div style="margin: 20px 0;font-size: 0.8em">

*Prepared for:* **Southwest Fox**  
*October 2018*

</div>

Security should be on every Web developer's mind when building a new Web application or enhancing an existing one. Not a day goes by that we don't hear about another security breach on some big Web site with scads of customer data compromised.

## Security is hard
Managing Web site security is never easy as there are a lot of different attack vectors and if you are new to Web development it's very easy to miss even simple security precautions. 

The good news is that the majority of security issues can be thwarted by a handful of good practices which I'll cover in this paper. But keep in mind that this is not all the things that can go wrong. I'm no security expert either but I've been around Web applications long enough to have seen most of the common attack vectors and know how to deal with them. But that's not to say that I have all the answers and this paper isn't meant to be an end all security document. If you are serious about security you should look at specific courses that deal explicitly with Web security, or even go as far as hiring a security specialist that can assess the state of security of your Web site.

Security is also an ongoing topic, something that needs to be kept up with. Attack vectors change over time, as do the tools you use to build and run your Web sites. 

The main takeaway from this short introduction is that Security is serious business and you should think about it right from the moment you start building your application, while you are adding new features and when it is up and running even when it is 'done'. Be vigilant.

## Web Connection and Security
West Wind Web Connection is a generic Web framework that provides an interface for FoxPro to interact with a Web Server - IIS Primarily - on Windows. Web Connection provides a rudimentary set of security features, but it is not and never was intended as a complete security solution. 

Part of this is because the majority of security related issues have little to do with the actual application itself and deal more with network management and IT administration. 

The focus of this paper is on the things that are important to a Web Connection application and that you as a developer using Web Connection and building a Web application have to think about.

Here's what I'm going to cover:

* Web Security
	* Web Server Security - IIS
	* TLS Encryption
	* Authentication
* Physical Access & Network
	* Who can get at the box?
	* Who can hack into the system
	* File system Permissions
	* Web Application Identity
* Operating System
	* Who can access files on the machine
* Middleware Technology
	* Who can hi-jack the application
	* Spoofing

## Web Server and Site Security
The first step is making sure that your Web Server and your Web Site are secure. Most of the issues around this are related to setup configuration of IIS and the specific Web site you are creating.

### IIS Security
The first line of defense involves the Web Server which in most cases for a Web Connection application will be Microsoft's built-in IIS server. IIS 7 and later is secure by default which means that when you install the Web Server it actually installs with very minimal features. The base install basically can serve static files and nothing more.

In order to configure IIS for Web applications and Web applications specifically you need to add a number of additional components that enable ASP.NET and or ISAPI, Authentication and some of the Administration features.

![](IISFeatures.png)

<small>**Figure 1** - Core features required for a Web Connection IIS installation</small>

The key things are:

* **ASP.NET or ISAPI**    
These are the connector interfaces that connects Web Connection to IIS. **You only need one or the other to run**, but both are supported in Web Connection and can be switched with simple configuration settings. The .NET module is the preferred mechanism which sports the most up to date code base and much more sophisticated administration features.

* **Authentication**  
In order to access the Web Connection administrative features and to perform the default admin folder blocking, Web Connection uses default Windows authentication. If you use .NET you only need to install Windows Authentication, but Basic Authentication can also be used. Both of these auth mechanisms are tied to Windows user accounts. Web Connection also provides application level security features that are separate from either of these mechanisms (more on that later).

* **IIS Management**    
In order for the Web Connection tools to configure IIS you need to have the IIS Management tools enabled, so you need to ensure the IIS Management Console is installed as well as the IIS 6 Metabase compatibility feature, which is a COM/ADS based IIS adminstration interface that's used by most tools.

### How IIS and Web Connection Interact
A key perspective for understanding IIS Web Security from an application execution perspective is to understand how IIS and your Web Connection use **Windows Identity** while the application is executing. 

#### It's all about Identity
The Windows Identity deteremines the rights that your application has on the Windows machine it is running on. A Web application's requests transfer through a number of Windows processes and each one has a specific Identity assigned to it. 

Identity is crucial to system security because it determines what your Web application can access on the local machine and potentially on the network. The higher the access rights, the higher risk that **if** your application is compromised that some damage can be inflicted on the entire machine. The key is is **if your application is compromised**.

There's an inverse relationship between how secure your application is and how much effort you have to put in to use more limited accounts. Using a high permissions account like SYSTEM or an Admin account lets your application freely access the entire machine, but it if there ever is a problem it also lets a hacker access your entire machine freely. If you choose to run under a more limited security scheme you have to **explicitly** expose each location on disk and the possibly the network that the application has access to.

Realize that clamping down security may not help you prevent access to data that your application uses anyway in case of an attack. Your application needs to have access, so in case of a security compromise that means a potential hacker also has access. Still, it's a good idea to minimize rights as much as possible by using a lower rights account and explicitly setting access where it's needed.

> #### @icon-warning Web Connection Uses SYSTEM by Default: Change it for Production
> When a new Web Connection application is created, Web Connection by default sets the Identity to SYSTEM which is a full access account on Windows. WWWC does this because SYSTEM is the only generic account in Windows that has the rights to **just work** out of the box when running in COM mode. Any other account requires some configuration. The setup routines are meant to configure a development machine initially and are not meant for production. For production choose a specific account, or NETWORK SERVICE and provide explicit system rights required by your application.


#### IIS and FoxPro
Let's drill in a little closer to understand where Identity applies. For IIS and Web Connection there are two different processes that you are concerned with and each can, but doesn't have to, have it's own process Identity:

* The IIS Application Pool
* Your FoxPro Exe Server

For Web Connection both are separate EXEs and each can have their own identity. 

> #### @icon-info-circle Use Launching User Passthrough Identity for FoxPro Server
> I recommend you **never explicitly set the identity of your FoxPro EXE** (in DCOMCnfg), but rather use the default, pass-through security of the **Launching User** that is used when custom DCOM Identity is applied. By doing so you only need to worry about the Identity of the Application Pool and not that of your FoxPro EXE.

#### The Process and Identity Hierachy
**Figure 2** shows the different processes that are involved when running an IIS Web Server:

![](IISWebConnectionHierarchy.png)

<small>**Figure 2** - IIS and Web Connection in the Context of Windows</small>

IIS is a Windows Web Server so everything is hosted within the context of the Windows OS. All the boxes you see in **Figure 2** are processes.

##### IIS Administration Service
The IIS Admin service is a top level service and somewhat disconnected system service that is responsible for launching Web Sites/Application Pools and monitoring them. When IIS starts or when you recycle IIS as a whole or an individual Application Pool you are interacting with the IIS Admin service. It's responsible for making sure that Web sites get started, keep running and monitors individual application pools for the various process limits you can configure in the Application Pool configuration. This service sits in the background of Windows and is internal to it - you generally don't configure or interact with it directly except when you use the Management Console, or **IISRESET** .

##### Application Pool
Application Pools are the **base process, an EXE** that one or more Web sites are hosted in. You can configure many application pools in IIS and you can add 1 or more Web sites to an application pool. Generally it's best to give mission critical applications their own application pool, while it's perfectly fine for many light use or static Web sites to be sharing a single Application Pool.

An application pool executes as an EXE: **w3wp.exe**. When you are running IIS and you have active clients you can see one or more w3wp.exe 

![](w3wpProcessExplorer.png)

<small>**Figure 3** - Application Pools (w3wp.exe) and Web Connection EXE are separate processes with their own identity </small>

I think of an Application Pool as the Web application and I like to set the Identity of the Application Pool in the Application Pool settings as the only place where Identity is set. Instead, I use the default passthrough security for any processes that are launched from the application pool.

![](ApplicationPoolIdentity.png)

<small>**Figure 4** - You can set Application Pool Identity in the Advanced Settings for the Pool</small>

##### FoxPro Web Connection Server
Your FoxPro Web Connection Server runs as a seperate **out of process COM EXE** server or as a file based standalone FoxPro application. 

File based servers **always** are started either as the Interactive User if you explicitly start the server from Explorer, or it is started using the Application Pools Identity.

COM servers use either the Application Pool's Identity - which I **highly recommend** - or the Identity you explicitly assign in DCOMCnfg. I really want to dissuade you from setting Identity in DcomCnfg simply because it can get very confusing what's running what. The only time that makes sense if you really want your IIS process and your FoxPro COM server to use different accounts.

The idea scenario is that the default DCOM Identity configuration is use which is the **Launching User** using **DcomCnfg**:

![](DcomCnfgIdentity.png)

<small>**Figure 5** - DcomCnfg lets you set the identity of your FoxPro server. Don't do it unless you have a good reason</small>

Note that **Figure 5** shows the default so you never have to explicitly set the **Launching User**. Only set this setting if you are changing the Identity to something else.

> Make sure to use the 32 bit version of DcomCnfg:  
> `MMC comexp.msc /32`

##### When do you need DcomCnfg?
One big frustration with Web Connection is that it runs EXE server that **might** need configuration. If you are using the default setup for Web Connection which uses the SYSTEM account **no DCOM Configuration is required**. 

No DCOM Configuration is required for:

* SYSTEM
* Administrator accounts
* Interactive

**All other accounts** have to configure the DCOM **Access and Launch** and **Activation** settings to allow specific users to launch either your COM server specifically or COM servers generically on the machine.

![](DcomConfigLaunchAndAccess.png)

<small>**Figure 6** - Setting Launch and Access and Activate for a DCOM Server</small>

These permissions can either be set on the specific server as shown here, or at the **Local Machine** level in which case they are applied to launching all EXE servers.  In this example, I'm explicitly adding **NETWORK SERVICE** to the the permissions. Both **Launch and Access** (shown) and **Activation** have to be set.

> #### @icon-info-circle Network Service as a Production Account
> For production machines I often use **Network Service** because its a low rights, generic account that has to be explicitly given access everywhere, but it's generic and doesn't require a password nor requires configuration of a special user account which makes your configuration more portable.

> #### @icon-warning Beware of the ApplicationPoolIdentity
> IIS by default creates new accounts using **ApplicationPoolIdentity** which is a dynamic account that has no rights on the local machine at all. You can't even set permissions for it in the Windows ACL dialogs. This account is meant for static sites that don't touch the local machine in any way and they are **not appropriate for use of Web Connection**. You will not be able to launch a COM Server or even a file server from the Web server with it.

#### Identity and your Application
Once your security is configured your application runs under a specific account and that account is what has access to the disk and other system services. If your app runs under NETWORK SERVICE, so you won't be able to write `HKEY_LOCAL_MACHINE` in the registry for example or write a file into the `c:\Windows\System32` directory.

The goal is to allow access only in application specific locations so that if your application is compromised in any way at worst you can damage your own application and the user can't take over the entire machine. If you run as SYSTEM, it is possible for the attacker to potentially plant malware or other executing code that monitors your machine and sends data off to somewhere else.

It all boils down to this:

> Choose an account to run your application that has the rights that your application needs to run and nothing more

### File System Security
Related to the process identity is File System security. The file system is probably the most vulnerable aspect when it comes to attempted hack attempts. Hackers love to exploit holes in applications that allow any sort of file uploads that might allow them to plant an executable in the file system, and the somehow execute that file to compromise security or access your data.

The best avenue to thwart that sort of scenario is to minimize file permissions as much as possible.

#### Choose a limited Application Account
A lot of this was discussed in the Application Pool security section where I discussed using a low rights account and then giving it just the rights needed to run the application. Once you have a low rights account start with minimal permissions needed and very selectively give WRITE permissions.

##### Web Folders
* Read/Execute Permissions in the Web Folder
* *Read/Write for `web.config` or `wc.ini` in Web Folder   
  to persist Admin page configuration settings (optional)*

##### Application Folder  
* Read/Execute in the Application/EXE folder
* Read/Write access in Data Folders
* Better: Don't use local data, but a SQL Backend for Data Access

#### Isolate your Data
In addition to system file access you also have to worry about data breaches. If you're using local or network FoxPro data you need to worry about those data locations.

##### Don't allow direct access from the Internet
This seems pretty obvious but any data you access from your application should only be accessible internally with no access from the Internet. Don't put data into a Web accessible path inside of your Web site. Always put data files into a completely separate non-Web accessible folder hierarchy.

Web Connection by default uses this structure:

```text
Project Root
--- Data                 Optional Data Folder
--- Deploy               Application Binaries
--- Web                  Mapped Web Folder
```

This is just a suggestion, but **whatever you do, never put data files (or anything else that is sensitive) into the Web folder**. It's acceptable to put data into the `Deploy` folder as a subfolder. Do put your data files into a self-contained folder so it's easy to move the data. 

And while you're at: For God's sake **don't hardcode paths in your application**. Try to always use Relative Paths, and if possible use variables for path names that can be read from a configuration file. If there's ever a problem being able to move the data quickly is key and having hard coded paths makes that very difficult. Configured paths from a configuration file can be changed without making code changes.

Ideally for security data should not be stored locally on the server, but rather sit on another machine that is not otherwise Internet accessible. The other machine should be on the internal network only or be accessible only via VPN. Make it so only your application account has access.

##### Use a SQL Backend on a Separate Server
An even better solution is to remove physical data entirely from the equation and instead storing your data inside of a SQL backend of some sort with the only way to access the data via username/password in the connection string that's encrypted.

As with data files, you want to make sure that the SQL backend is not exposed directly to the Internet. SQL Server by default doesn't allow remote access, but you can lock it down to specify which IP addresses or subnets have access. Likewise databases like MongoDb let you cut off internet access completely. Either way make sure that you use complex username and password sequences that are hard to break and store passwords in a safe place - encrypted if possible.

## Web Server 







### Network Security