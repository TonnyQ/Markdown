# Developing for Windows10 UWP APPs
## 1.Setting Up Your Environment
* First,to build Windows UWP apps,you need a Window10 OS;
* Second,Visual Studio Community Edition 2015 or Professional;
* Finally,Windows 10 SDK(Standalone).
## 2.Creating Your First Project
* First,Select **File->New->Project** from the main menu to open up the VS.
* Second,using **Visual C#** as the language of choice and select **Windows->Universal** item;
* Third,Select **Blank App(Universal Windows)** and give the project the name and location on your workstation;
* Select **OK** to generate the project and Open **develop mode** in **Setting->Update&security->Developing**.
## 3.Exploring the App Manifest
**Package.appxmanifest**:This is the manifest for the app.double-click it to open it up;The app manifest consists of six tabs grouped by the function they server in helping define your app to the Windows Store and end user machines that install it.
* Application Tab Fields:
    1. **Display Name**:The full name of the app.the name that will be displayed on the Application title bar and will server as the "name" of the app.
    2. **Entry Point**:to identify the class that runs when the application is started.
    3. **Description**:Description for the app.The text is used in the "Set Default Programs" Windows 10 UI.
    4. **Supported Rotations**:identies the orientations supported by an app.
    5. **Recurrence**:the system how often the URI specified in the URI Template field should be polled.
    6. **URI Template**:specify a URI where a Template used to render a given app's tile should be pulled from.The template are XML snippets that the Windows 10 system uses to determine what to display on a given tile with what layout and animation.
* Visual Assets tab:
    Contains fields for populating the locations fot all the standard visual assets needed to display your application in the store and on an end user's device.This includes all sizes of application logos and other app-specific iconography,and the splash screen image.
* Capabilities tab:
    1. show all the things app can do;
    2. combined with the sandboxed nature of UWP apps,keeps the end user in control of the resources app can access.
    3. as a developer,must explicitly declared a Capabilities in the manifest.
    4. [itemizes of the Capabilities](https://msdn.microsoft.com/en-us/library/windows/apps/Hh464936.aspx);
* Content URIs tab:
    allow to specify URIs that can use the **Window.external.notify** JavaScript function to send a **ScriptNotify** event to the app.
* Packaging tab:can be used to specify the properties that identify and describe your app package when it is deployed.
    1. **Package Name**:Unique name that identifies the package on the development/test system.
    2. **Package Display Name**:app name that appears the Windows Store,also replace when deployed.
    3. **Version**:version string following the format **<major>.<minor>.<build>.<revision>**;
    4. **Publisher**:Temporarily represents the subject field of the certificate used to sign app package.
    5. **Publisher Display Name**:Name from the Publisher name of the develop web portal.
    6. **Package Family**:Unique name that identifies the package on the system.
