21\. Introducción y enumeración de Active Directory
===================================================

En este módulo de aprendizaje, cubriremos las siguientes unidades de aprendizaje:

*   Introducción a Active Directory
*   Enumeración de Active Directory mediante herramientas manuales
*   Enumeración de Active Directory mediante herramientas automatizadas

[_Active Directory Domain Services_](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) , a menudo denominado Active Directory (AD), es un servicio que permite a los administradores de sistemas actualizar y administrar sistemas operativos, aplicaciones, usuarios y acceso a datos a gran escala. Active Directory se instala con una configuración estándar, sin embargo, los administradores de sistemas a menudo lo personalizan para que se ajuste a las necesidades de la organización.

Desde la perspectiva de un evaluador de penetración, Active Directory es muy interesante ya que normalmente contiene una gran cantidad de información. Si logramos comprometer ciertos objetos dentro del dominio, podremos tomar el control total de la infraestructura de la organización.

En este módulo de aprendizaje, nos centraremos en el aspecto de enumeración de Active Directory. La información que recopilaremos a lo largo del módulo tendrá un impacto directo en los distintos ataques que realizaremos en el próximo módulo _Ataques a la autenticación de Active Directory_ y _al movimiento lateral en_ módulos de Active Directory.

21.1. Active Directory - Introducción
-------------------------------------

Esta unidad de aprendizaje cubrirá los siguientes objetivos de aprendizaje:

*   Introducción a Active Directory
*   Definir nuestros objetivos de enumeración

Si bien Active Directory es en sí mismo un servicio, también actúa como una capa de administración. AD contiene información crítica sobre el entorno y almacena información sobre usuarios, grupos y equipos, cada uno de los cuales se denomina _objetos_ . Los permisos establecidos en cada objeto determinan los privilegios que ese objeto tiene dentro del dominio.

Configurar y mantener una instancia de Active Directory puede resultar desalentador para los administradores, especialmente porque la gran cantidad de información contenida a menudo crea una gran superficie de ataque.

El primer paso para configurar una instancia de AD es crear un nombre de dominio como _corp.com_ , en el que _corp_ suele ser el nombre de la organización. Dentro de este dominio, los administradores pueden agregar varios tipos de objetos asociados con la organización, como computadoras, usuarios y objetos de grupo.

Un entorno de AD tiene una dependencia crítica del servicio del _Sistema de nombres de dominio_ (DNS). Por ello, un controlador de dominio típico también alojará un servidor DNS que tenga autoridad para un dominio determinado.

Para facilitar la gestión de diversos objetos y ayudar con la administración, los administradores de sistemas a menudo organizan estos objetos en [_unidades organizativas_](https://en.wikipedia.org/wiki/Organizational_unit_(computing)) (OU).

Las OU son comparables a las carpetas del sistema de archivos en el sentido de que son contenedores que se utilizan para almacenar objetos dentro del dominio. Los objetos de equipo representan servidores y estaciones de trabajo reales que están unidos al dominio (parte del dominio), y los objetos de usuario representan cuentas que se pueden utilizar para iniciar sesión en los equipos unidos al dominio. Además, todos los objetos de AD contienen atributos, que variarán según el tipo de objeto. Por ejemplo, un objeto de usuario puede incluir atributos como nombre, apellido, nombre de usuario, número de teléfono, etc.

AD depende de varios componentes y servicios de comunicación. Por ejemplo, cuando un usuario intenta iniciar sesión en el dominio, se envía una solicitud a un [_controlador de dominio_](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-authsod/c4012a57-16a9-42eb-8f64-aa9e04698dca) (DC), que verifica si el usuario tiene permiso o no para iniciar sesión en el dominio. Uno o más controladores de dominio actúan como el centro y núcleo del dominio, almacenando todas las unidades organizativas, los objetos y sus atributos. Dado que el controlador de dominio es un componente central del dominio, le prestaremos mucha atención a medida que enumeramos AD.

Los objetos se pueden asignar a grupos de AD para que los administradores puedan gestionar esos objetos como una sola unidad. Por ejemplo, a los usuarios de un grupo se les puede dar acceso a un recurso compartido de servidor de archivos o acceso administrativo a varios clientes del dominio. Los atacantes suelen dirigirse a grupos con privilegios elevados.

Los miembros del grupo de [_administradores de dominio_](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#domain-admins) se encuentran entre los objetos más privilegiados del dominio. Si un atacante pone en peligro a un miembro de este grupo (a menudo denominados _administradores de dominio_ ), básicamente obtiene el control total del dominio.

Este vector de ataque podría extenderse más allá de un solo dominio, ya que una instancia de AD puede alojar más de un dominio en un _árbol de dominios_ o varios árboles de dominios en un _bosque de dominios_ . Si bien existe un grupo de administradores de dominio para cada dominio del bosque, a los miembros del grupo _de administradores de empresa_ se les otorga control total sobre todos los dominios del bosque y tienen privilegios de administrador en todos los controladores de dominio. Obviamente, este es un objetivo de alto valor para un atacante.

En este módulo, aprovecharemos estos y otros conceptos mientras nos centramos en el aspecto extremadamente importante de la enumeración de AD. Esta importante disciplina puede mejorar nuestro éxito durante la fase de ataque. Aprovecharemos una variedad de herramientas para enumerar manualmente AD, la mayoría de las cuales se basan en el [_Protocolo ligero de acceso a directorios_](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol) (LDAP). Una vez que hayamos presentado las técnicas fundamentales, aprovecharemos la automatización para realizar la enumeración a escala.

21.1.1. Enumeración: definición de nuestros objetivos
-----------------------------------------------------

Antes de comenzar, analicemos el escenario y definamos nuestros objetivos.

En este escenario, enumeraremos el dominio _corp.com_ . Hemos obtenido las credenciales de usuario de un usuario de dominio a través de un ataque de phishing exitoso. Alternativamente, la organización objetivo puede habernos proporcionado las credenciales de usuario para que podamos realizar pruebas de penetración basadas en una _supuesta infracción_ . Esto aceleraría el proceso para nosotros y también le daría a la organización una idea de la facilidad con la que los atacantes pueden moverse dentro de su entorno una vez que hayan obtenido el acceso inicial.

El usuario al que tenemos acceso es _Stephanie_ , que tiene permisos de escritorio remoto en una máquina con Windows 11 que forma parte del dominio. Este usuario no es un administrador local en la máquina, algo que tal vez debamos tener en cuenta a medida que avanzamos.

Durante una evaluación en el mundo real, la organización también puede definir el alcance y los objetivos de la prueba de penetración. Sin embargo, en nuestro caso, estamos restringidos al dominio _corp.com_ con los laboratorios PWK. Nuestro objetivo será enumerar el dominio completo, incluida la búsqueda de posibles formas de lograr el mayor privilegio posible (administrador de dominio en este caso).

En este módulo, realizaremos la enumeración desde una máquina cliente con el usuario de dominio _stephanie_ con privilegios bajos . Sin embargo, una vez que comencemos a realizar ataques y podamos obtener acceso a usuarios y computadoras adicionales, es posible que tengamos que repetir partes del proceso de enumeración desde el nuevo punto de vista. Este cambio de perspectiva (o _pivote_ ) es fundamental durante el proceso de enumeración considerando la complejidad de los permisos en todo el dominio. Cada pivote puede brindarnos una oportunidad para avanzar en nuestro ataque.

Por ejemplo, si obtenemos acceso a otra cuenta de usuario con pocos privilegios que parece tener el mismo acceso que _stephanie_ , no deberíamos simplemente descartarla. En cambio, siempre deberíamos repetir nuestra enumeración con esa nueva cuenta, ya que los administradores a menudo otorgan a los usuarios individuales mayores permisos en función de su rol exclusivo en la organización. Este proceso persistente de "enjuagar y repetir" es la clave para una enumeración exitosa y funciona muy bien, especialmente en organizaciones grandes.

21.2. Active Directory: enumeración manual
------------------------------------------

Esta unidad de aprendizaje cubrirá los siguientes objetivos de aprendizaje:

*   Enumerar Active Directory utilizando aplicaciones heredadas de Windows
*   Utilice PowerShell y .NET para realizar enumeraciones de AD adicionales

Existen muchas formas de enumerar AD y una amplia variedad de herramientas que podemos utilizar. En esta unidad de aprendizaje, comenzaremos a enumerar el dominio utilizando herramientas que ya están instaladas en Windows. Comenzaremos con la información más sencilla, es decir, la que podemos recopilar de forma rápida y sencilla. Finalmente, aprovecharemos técnicas más sólidas, como invocar clases .NET mediante PowerShell para comunicarnos con AD a través de LDAP.

21.2.1 Active Directory: enumeración mediante herramientas heredadas de Windows
-------------------------------------------------------------------------------

Dado que comenzamos con un _supuesto_ escenario de violación y tenemos credenciales para _stephanie_ , usaremos esas credenciales para autenticarnos en el dominio a través de una máquina con Windows 11 (CLIENT75). Usaremos el _Protocolo de escritorio remoto_ (RDP) con _xfreerdp_ para conectarnos al cliente e iniciar sesión en el dominio. Proporcionaremos el nombre de usuario con **/u** , el nombre de dominio con **/d** e ingresaremos la contraseña, que en este caso es _LegmanTeamBenzoin!!_ .

    kali@kali:~$ xfreerdp /u:stephanie /d:corp.com /v:192.168.50.75
    

> Listado 1 - Conexión al cliente de Windows 11 mediante "xfreerdp"

AD contiene tanta información que puede resultar difícil determinar por dónde empezar a enumerarla. Pero como cada instalación de AD contiene básicamente usuarios y grupos, comenzaremos por ahí.

Para comenzar a recopilar información de los usuarios, utilizaremos [**net.exe**](https://learn.microsoft.com/en-US/troubleshoot/windows-server/networking/net-commands-on-operating-systems) , que se instala de forma predeterminada en todos los sistemas operativos Windows. Más específicamente, utilizaremos el subcomando [**net user**](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/cc771865(v=ws.11)?redirectedfrom=MSDN) . Si bien podemos utilizar esta herramienta para enumerar las cuentas locales en la máquina, utilizaremos **/domain** para imprimir los usuarios del dominio.

    C:\Users\stephanie>net user /domain
    The request will be processed at a domain controller for domain corp.com.
    
    User accounts for \\DC1.corp.com
    
    -------------------------------------------------------------------------------
    Administrator            dave                     Guest
    iis_service              jeff                     jeffadmin
    jen                      krbtgt                   pete
    stephanie
    The command completed successfully.
    

> Listado 2 - Ejecución de "net user" para mostrar los usuarios del dominio

El resultado de este comando variará según el tamaño de la organización. Con una lista de usuarios, ahora podemos consultar información sobre usuarios individuales.

Los administradores suelen tener tendencia a agregar prefijos o sufijos a los nombres de usuario que identifican las cuentas por su función. Según el resultado del Listado 2, deberíamos verificar el usuario _jeffadmin_ porque podría ser una cuenta administrativa.

Inspeccionemos el usuario con **net.exe** y el indicador **/domain** :

    C:\Users\stephanie>net user jeffadmin /domain
    The request will be processed at a domain controller for domain corp.com.
    
    User name                    jeffadmin
    Full Name
    Comment
    User's comment
    Country/region code          000 (System Default)
    Account active               Yes
    Account expires              Never
    
    Password last set            9/2/2022 4:26:48 PM
    Password expires             Never
    Password changeable          9/3/2022 4:26:48 PM
    Password required            Yes
    User may change password     Yes
    
    Workstations allowed         All
    Logon script
    User profile
    Home directory
    Last logon                   9/20/2022 1:36:09 AM
    
    Logon hours allowed          All
    
    Local Group Memberships      *Administrators
    Global Group memberships     *Domain Users         *Domain Admins
    The command completed successfully.
    

> Listado 3 - Ejecución de "net user" contra un usuario específico

Según el resultado, _jeffadmin_ forma parte del grupo _de administradores de dominio_ , algo que debemos tener en cuenta. Si logramos poner en riesgo esta cuenta, básicamente nos convertiremos en administradores de dominio.

También podemos usar **net.exe** para enumerar grupos en el dominio con [**net group**](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc754051(v=ws.10)?redirectedfrom=MSDN) :

    C:\Users\stephanie>net group /domain
    The request will be processed at a domain controller for domain corp.com.
    
    Group Accounts for \\DC1.corp.com
    
    -------------------------------------------------------------------------------
    *Cloneable Domain Controllers
    *Debug
    *Development Department
    *DnsUpdateProxy
    *Domain Admins
    *Domain Computers
    *Domain Controllers
    *Domain Guests
    *Domain Users
    *Enterprise Admins
    *Enterprise Key Admins
    *Enterprise Read-only Domain Controllers
    *Group Policy Creator Owners
    *Key Admins
    *Management Department
    *Protected Users
    *Read-only Domain Controllers
    *Sales Department
    *Schema Admins
    The command completed successfully.
    

> Listado 4 - Ejecución de "net group" para mostrar los grupos en el dominio

El resultado incluye una larga lista de grupos en el dominio. Algunos de ellos están [instalados de forma predeterminada](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups) . Otros, como los resaltados anteriormente, son grupos personalizados creados por el administrador. Enumeremos primero un grupo personalizado.

Utilizaremos nuevamente **net.exe** para enumerar los miembros del grupo, esta vez centrándonos en el grupo _del Departamento de Ventas_ .

    PS C:\Tools> net group "Sales Department" /domain
    The request will be processed at a domain controller for domain corp.com.
    
    Group name     Sales Department
    Comment
    
    Members
    
    -------------------------------------------------------------------------------
    pete                     stephanie
    The command completed successfully.
    

> Listado 5 - Ejecución de "net group" para mostrar los miembros de un grupo específico

Esto revela que _Pete_ y _Stephanie_ son miembros del grupo del _Departamento de Ventas_ .

Aunque esto no parece revelar mucho, cada pequeña pieza de información obtenida mediante la enumeración es potencialmente valiosa. En una evaluación del mundo real, podríamos enumerar cada grupo y catalogar los resultados. Esto requerirá una buena organización, que analizaremos más adelante, pero continuaremos por ahora, ya que tenemos alternativas más flexibles a net.exe que analizaremos en la siguiente sección.

Recursos
--------

Algunos de los laboratorios requieren que inicie las máquinas de destino que se indican a continuación.

Tenga en cuenta que las direcciones IP asignadas a sus máquinas de destino pueden no coincidir con las referenciadas en el texto y el video del módulo.

Nombre

(Haga clic para ordenar en orden ascendente)

Dirección IP

Enumeración de Active Directory: enumeración con herramientas heredadas de Windows: grupo de máquinas virtuales 1

Iniciar **la enumeración de Active Directory: enumeración mediante herramientas heredadas de Windows: grupo de máquinas virtuales 1** con acceso al navegador Kali

Enumeración de Active Directory: enumeración con herramientas heredadas de Windows: grupo de máquinas virtuales 2

Iniciar **la enumeración de Active Directory: enumeración mediante herramientas heredadas de Windows: grupo de máquinas virtuales 2** con acceso al navegador Kali

#### Laboratorios

1.  ¿Qué tipo de servidor actúa como núcleo y concentrador de un dominio alojado en Active Directory?

Respuesta

Verificar

2.  Inicie el grupo de máquinas virtuales 1 e inicie sesión en CLIENT75 como _stephanie_ . Utilice **net.exe** para enumerar el dominio _corp.com_ . ¿Qué usuario es miembro del grupo del _Departamento de administración ?_

Respuesta[Ver sugerencias](#)

Verificar

3.  Inicie el grupo de máquinas virtuales 2 e inicie sesión en CLIENT75 como _stephanie_ . Utilice **net.exe para enumerar los usuarios y grupos en el dominio** _corp.com_ modificado para obtener la marca.

Respuesta[Ver sugerencias](#)

Verificar

21.2.2. Enumeración de Active Directory mediante PowerShell y clases .NET
-------------------------------------------------------------------------

Existen varias herramientas que podemos utilizar para enumerar Active Directory. Los cmdlets de PowerShell como [_Get-ADUser_](https://learn.microsoft.com/en-us/powershell/module/activedirectory/get-aduser?view=windowsserver2022-ps) funcionan bien, pero solo se instalan de forma predeterminada en los controladores de dominio como parte de las [_Herramientas de administración remota del servidor_](https://learn.microsoft.com/en-us/troubleshoot/windows-server/system-management-components/remote-server-administration-tools) (RSAT). Las RSAT rara vez están presentes en los clientes de un dominio y debemos tener privilegios administrativos para instalarlas. Si bien, en principio, podemos importar nosotros mismos la DLL necesaria para la enumeración, analizaremos otras opciones.

Desarrollaremos una herramienta que solo requiere privilegios básicos y es lo suficientemente flexible para usarla en interacciones del mundo real. Imitaremos las consultas que se producen como parte del funcionamiento habitual de AD. Esto nos ayudará a comprender los conceptos básicos que se utilizan en las herramientas prediseñadas que utilizaremos más adelante.

En concreto, utilizaremos PowerShell y clases .NET para crear un script que enumere el dominio. Aunque el desarrollo de PowerShell puede parecer complejo, lo haremos paso a paso.

Para enumerar AD, primero debemos entender cómo comunicarnos con el servicio. Antes de comenzar a crear nuestro script, analicemos un poco la teoría.

La enumeración de AD se basa en LDAP. Cuando una máquina de dominio busca un objeto, como una impresora, o cuando consultamos objetos de usuario o grupo, se utiliza LDAP como canal de comunicación para la consulta. En otras palabras, LDAP es el protocolo utilizado para comunicarse con Active Directory.

LDAP no es exclusivo de AD. Otros servicios de directorio también lo utilizan.

La comunicación LDAP con AD no siempre es sencilla, pero aprovecharemos una [_Interfaz de servicios de directorio activo_](https://learn.microsoft.com/en-us/windows/win32/adsi/active-directory-service-interfaces-adsi) (ADSI) (un conjunto de interfaces creadas en [_COM_](https://learn.microsoft.com/en-us/windows/win32/com/com-objects-and-interfaces) ) como proveedor LDAP.

Según [la documentación de Microsoft](https://learn.microsoft.com/en-us/windows/win32/adsi/ldap-adspath?redirectedfrom=MSDN) , necesitamos una ruta LDAP _ADsPath_ específica para comunicarnos con el servicio AD. El prototipo de la ruta LDAP se ve así:

    LDAP://HostName[:PortNumber][/DistinguishedName]
    

> Listado 6 - Formato de ruta LDAP

Necesitamos tres parámetros para una ruta LDAP completa: _HostName_ , _PortNumber_ y _DistinguishedName_ . Dediquemos un momento a analizar esto.

El _nombre de host_ puede ser un nombre de computadora, una dirección IP o un nombre de dominio. En nuestro caso, estamos trabajando con el dominio _corp.com_ , por lo que podríamos simplemente agregarlo a nuestra ruta LDAP y probablemente obtener información. Tenga en cuenta que un dominio puede tener varios controladores de dominio, por lo que configurar el nombre de dominio podría potencialmente resolver la dirección IP de cualquier controlador de dominio en el dominio.

Si bien es probable que esto aún devuelva información válida, puede que no sea el enfoque de enumeración más óptimo. De hecho, para que nuestra enumeración sea lo más precisa posible, debemos buscar el controlador de dominio que contenga la información más actualizada. Esto se conoce como el [_controlador de dominio principal_](https://learn.microsoft.com/en-GB/troubleshoot/windows-server/identity/fsmo-roles) (PDC). Solo puede haber un PDC en un dominio. Para encontrar el PDC, debemos encontrar el controlador de dominio que contiene la propiedad _PdcRoleOwner_ . Finalmente, usaremos PowerShell y una clase .NET específica para encontrarlo.

El _número de puerto_ para la conexión LDAP es opcional según la documentación de Microsoft. En nuestro caso, no agregaremos el número de puerto, ya que elegirá automáticamente el puerto en función de si estamos utilizando o no una conexión SSL. Sin embargo, vale la pena señalar que si en el futuro nos encontramos con un dominio que utiliza puertos no predeterminados, es posible que debamos agregarlo manualmente al script.

Por último, un [_nombre distintivo_](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/distinguished-names) (DN) es una parte de la ruta LDAP. Un DN es un nombre que identifica de forma única un objeto en AD, incluido el dominio en sí. Si no estamos familiarizados con LDAP, esto puede resultar un poco confuso, así que analicemos un poco más en detalle.

Para que LDAP funcione, los objetos en AD (u otros servicios de directorio) deben formatearse de acuerdo con un [estándar de nombres](https://www.rfc-editor.org/rfc/rfc2247.html) específico . Para mostrar un ejemplo de un DN, podemos usar nuestro usuario de dominio _stephanie_ . Sabemos que _stephanie_ es un objeto de usuario dentro del dominio _corp.com_ . Con esto, el DN puede (aunque no podemos estar seguros todavía) verse así:

    CN=Stephanie,CN=Users,DC=corp,DC=com
    

> Listado 7 - Ejemplo de un nombre distinguido

La lista anterior muestra algunas referencias nuevas que no hemos visto antes en este módulo, como _CN_ y _DC_ . El CN se conoce como el _nombre común_ , que especifica el identificador de un objeto en el dominio. Si bien normalmente nos referimos a "DC" como el controlador de dominio en términos de AD, "DC" significa _componente de dominio_ cuando nos referimos a un nombre distinguido. El _componente de dominio_ representa la parte superior de un árbol LDAP y, en este caso, nos referimos a él como el nombre distinguido del dominio en sí.

Al leer un DN, comenzamos con los objetos de componente de dominio en el lado derecho y nos desplazamos hacia la izquierda. En el ejemplo anterior, tenemos cuatro componentes, comenzando con dos componentes denominados _DC=corp,DC=com_ . Los objetos de componente de dominio, como se mencionó anteriormente, representan la parte superior de un árbol LDAP siguiendo el estándar de nombres requerido.

Continuando con el DN, _CN=Users_ representa el nombre común del contenedor donde se almacena el objeto de usuario (también conocido como contenedor principal). Finalmente, a la izquierda, _CN=Stephanie_ representa el nombre común del objeto de usuario en sí, que también se encuentra en el nivel más bajo de la jerarquía.

En nuestro caso, para la ruta LDAP, nos interesa el objeto Componente de dominio, que es _DC=corp,DC=com_ . Si añadiéramos _CN=Users_ a nuestra ruta LDAP, nos limitaríamos a poder buscar únicamente objetos dentro de ese contenedor determinado.

Comencemos a escribir nuestro script obteniendo el nombre de host requerido para el PDC.

En las [clases de Microsoft .NET](https://learn.microsoft.com/en-us/dotnet/api/) relacionadas con AD, encontramos el espacio de nombres _System.DirectoryServices.ActiveDirectory_ . Si bien hay algunas clases para elegir aquí, nos centraremos en la [_clase Domain_](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.activedirectory.domain?view=windowsdesktop-7.0) . Contiene específicamente una referencia a _PdcRoleOwner_ en las propiedades, que es exactamente lo que necesitamos. Al verificar los métodos, encontramos un método llamado _GetCurrentDomain()_ , que devolverá el objeto de dominio para el usuario actual, en este caso _stephanie_ .

Para invocar la _clase de dominio_ y el método _GetCurrentDomain_ , ejecutaremos el siguiente comando en PowerShell:

    PS C:\Users\stephanie> [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
    
    Forest                  : corp.com
    DomainControllers       : {DC1.corp.com}
    Children                : {}
    DomainMode              : Unknown
    DomainModeLevel         : 7
    Parent                  :
    PdcRoleOwner        : DC1.corp.com
    RidRoleOwner            : DC1.corp.com
    InfrastructureRoleOwner : DC1.corp.com
    Name                  	: corp.com
    

> Listado 8 - Clase de dominio del espacio de nombres System.DirectoryServices.ActiveDirectory

El resultado revela la propiedad _PdcRoleOwner_ , que en este caso es _DC1.corp.com_ . Si bien podemos agregar este nombre de host directamente en nuestro script como parte de la ruta LDAP, queremos automatizar el proceso para poder usar también este script en futuras interacciones.

Hagamos esto paso a paso. Primero, crearemos una variable que almacenará el objeto de dominio y luego imprimiremos la variable para poder verificar que aún funciona dentro de nuestro script. La primera parte de nuestro script se muestra a continuación:

    # Store the domain object in the $domainObj variable
    $domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
    
    # Print the variable
    $domainObj
    

> Listado 9 – Almacenamiento del objeto de dominio en nuestra primera variable

Para ejecutar el script, debemos omitir la política de ejecución, que fue diseñada para evitar que ejecutemos scripts de PowerShell por accidente. Lo haremos con **powershell -ep bypass** :

    PS C:\Users\stephanie> powershell -ep bypass
    Windows PowerShell
    Copyright (C) Microsoft Corporation. All rights reserved.
    
    Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows
    
    PS C:\Users\stephanie>
    

Ahora ejecutemos nuestro script y verifiquemos que imprime el objeto de dominio:

    PS C:\Users\stephanie> .\enumeration.ps1
    
    Forest                  : corp.com
    DomainControllers       : {DC1.corp.com}
    Children                : {}
    DomainMode              : Unknown
    DomainModeLevel         : 7
    Parent                  :
    PdcRoleOwner            : DC1.corp.com
    RidRoleOwner            : DC1.corp.com
    InfrastructureRoleOwner : DC1.corp.com
    Name                    : corp.com
    

> Listado 10 - Salida que muestra la información almacenada en nuestra primera variable

Nuestra variable _domainObj_ ahora contiene la información sobre el objeto de dominio. Aunque esta declaración de impresión no es obligatoria, es una buena manera de verificar que nuestro comando y la variable funcionaron como estaba previsto.

Dado que el nombre de host en la propiedad _PdcRoleOwner_ es necesario para nuestra ruta LDAP, podemos extraer el nombre directamente del objeto de dominio. En caso de que necesitemos más información del objeto de dominio más adelante en nuestro script, mantendremos $ _domainObj_ por el momento y crearemos una nueva variable llamada _$PDC_ , que extraerá el valor de la propiedad _PdcRoleOwner_ contenida en nuestra variable _$domainObj :_

    # Store the domain object in the $domainObj variable
    $domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
    
    # Store the PdcRoleOwner name to the $PDC variable
    $PDC = $domainObj.PdcRoleOwner.Name
    
    # Print the $PDC variable
    $PDC
    

> Listado 11 - Agregar la variable $PDC a nuestro script y extraer el nombre PdcRoleOwner

Ahora ejecutemos el script nuevamente e inspeccionemos el resultado:

    PS C:\Users\stephanie> .\enumeration.ps1
    DC1.corp.com
    

> Listado 12 - Impresión de la variable $PDC

En este caso, hemos extraído dinámicamente el PDC de la propiedad _PdcRoleOwner_ mediante la clase Domain. Bien.

Si bien también podemos obtener el DN del dominio a través del objeto de dominio, no sigue el estándar de nombres requerido por LDAP. En nuestro ejemplo, sabemos que el dominio base es _corp.com_ y el DN sería, de hecho, _DC=corp,DC=com_ . En este caso, podríamos tomar _corp.com_ de la propiedad _Name_ en el objeto de dominio y decirle a PowerShell que lo divida y agregue el parámetro _DC=_ requerido . Sin embargo, hay una forma más sencilla de hacerlo, que también garantizará que obtengamos el DN correcto.

Podemos usar ADSI directamente en PowerShell para recuperar el DN. Usaremos dos comillas simples para indicar que la búsqueda comienza en la parte superior de la jerarquía de AD.

    PS C:\Users\stephanie> ([adsi]'').distinguishedName
    DC=corp,DC=com
    

> Listado 13 - Uso de ADSI para obtener el DN del dominio

Esto devuelve el DN en el formato adecuado para la ruta LDAP.

Ahora podemos agregar una nueva variable en nuestro script que almacenará el DN del dominio. Para asegurarnos de que el script aún funciona, agregaremos una declaración _de impresión_ e imprimiremos el contenido de nuestra nueva variable:

    # Store the domain object in the $domainObj variable
    $domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
    
    # Store the PdcRoleOwner name to the $PDC variable
    $PDC = $domainObj.PdcRoleOwner.Name
    
    # Store the Distinguished Name variable into the $DN variable
    $DN = ([adsi]'').distinguishedName
    
    # Print the $DN variable
    $DN
    

> Listado 14 - Creación de una nueva variable que contiene el DN del dominio

Ejecutemos el script.

    PS C:\Users\stephanie> .\enumeration.ps1
    DC=corp,DC=com
    

> Listado 15 – Uso de nuestro script para imprimir el DN del dominio

En este punto, estamos obteniendo dinámicamente el nombre de host y el DN con nuestro script. Ahora debemos ensamblar las piezas para construir la ruta LDAP completa. Para ello, agregaremos una nueva variable _$LDAP_ a nuestro script que contendrá las variables _$PDC_ y _$DN_ , con el prefijo "LDAP://".

El script final genera el LDAP que se muestra a continuación. Tenga en cuenta que, para limpiarlo, hemos eliminado los comentarios. Dado que solo necesitábamos el valor del nombre de la propiedad _PdcRoleOwner_ del objeto de dominio, lo agregamos directamente en nuestra variable _$PDC_ en la primera línea, lo que limita la cantidad de código necesario:

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName 
    $LDAP = "LDAP://$PDC/$DN"
    $LDAP
    

> Listado 16: secuencia de comandos que creará la ruta LDAP completa requerida para la enumeración

Ejecutemos el script.

    PS C:\Users\stephanie> .\enumeration.ps1
    LDAP://DC1.corp.com/DC=corp,DC=com
    

> Listado 17 - Salida de script que muestra la ruta LDAP completa

¡Genial! Hemos utilizado con éxito clases .NET y ADSI para obtener de forma dinámica la ruta LDAP completa necesaria para nuestra enumeración. Además, nuestro script es dinámico, por lo que podemos reutilizarlo fácilmente en interacciones del mundo real.

Recursos
--------

Algunos de los laboratorios requieren que inicie las máquinas de destino que se indican a continuación.

Tenga en cuenta que las direcciones IP asignadas a sus máquinas de destino pueden no coincidir con las referenciadas en el texto y el video del módulo.

Nombre

(Haga clic para ordenar en orden ascendente)

Dirección IP

Enumeración de Active Directory: enumeración de Active Directory mediante PowerShell (grupo de máquinas virtuales 1)

Iniciar **la enumeración de Active Directory: enumeración de Active Directory mediante PowerShell: grupo de máquinas virtuales 1** con acceso al navegador Kali

#### Laboratorios

1.  Inicie el grupo de máquinas virtuales 1 y repita los pasos descritos en esta sección para crear el script. Utilice el script para obtener dinámicamente la ruta LDAP para el dominio _corp.com_ . ¿Qué propiedad del _objeto de dominio_ muestra el controlador de dominio principal del dominio?

Respuesta[Ver sugerencias](#)

Verificar

2.  ¿Qué conjunto de interfaces COM nos proporciona un proveedor LDAP que podemos usar para comunicarnos con Active Directory?

Respuesta

Verificar

21.2.3. Añadiendo funcionalidad de búsqueda a nuestro script
------------------------------------------------------------

Hasta ahora, nuestro script crea la ruta LDAP requerida. Ahora podemos incorporar la función de búsqueda.

Para ello, utilizaremos dos clases .NET que se encuentran en el espacio de nombres _System.DirectoryServices_ , más específicamente las clases [_DirectoryEntry_](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.directoryentry?view=dotnet-plat-ext-6.0) y [_DirectorySearcher_](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.directorysearcher?view=dotnet-plat-ext-6.0) . Analicemos estas clases antes de implementarlas.

La clase _DirectoryEntry_ encapsula un objeto en la jerarquía de servicios de AD. En nuestro caso, queremos buscar desde la parte superior de la jerarquía de AD, por lo que proporcionaremos la ruta LDAP obtenida a la clase _DirectoryEntry_ .

Algo que hay que tener en cuenta con _DirectoryEntry_ es que podemos pasarle credenciales para autenticarnos en el dominio. Sin embargo, como ya hemos iniciado sesión, no es necesario hacerlo aquí.

La clase _DirectorySearcher_ realiza consultas en AD mediante LDAP. Al crear una instancia de _DirectorySearcher_ , debemos especificar el servicio de AD que queremos consultar en forma de propiedad [_SearchRoot_](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.directorysearcher.searchroot?view=dotnet-plat-ext-6.0) . Según la documentación de Microsoft, esta propiedad indica dónde comienza la búsqueda en la jerarquía de AD. Dado que la clase _DirectoryEntry_ encapsula la ruta LDAP que apunta a la parte superior de la jerarquía, la pasaremos como una variable a _DirectorySearcher_ .

La documentación _de DirectorySearcher enumera_ [_FindAll()_](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.directorysearcher.findall?view=dotnet-plat-ext-7.0#system-directoryservices-directorysearcher-findall) , que devuelve una colección de todas las entradas encontradas en AD.

Implementemos estas dos clases en nuestro script. El código a continuación muestra la parte relevante del script:

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName 
    $LDAP = "LDAP://$PDC/$DN"
    
    $direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
    
    $dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
    $dirsearcher.FindAll()
    

> Listado 18 - Directorio y DirectorySearcher a nuestro script

Como se indica en el Listado 18, hemos agregado la variable _$direntry_ , que encapsula la ruta LDAP obtenida. La variable _$dirsearcher_ contiene la variable _$direntry_ y utiliza la información como _SearchRoot_ , que apunta a la parte superior de la jerarquía donde _DirectorySearcher_ ejecutará el método _FindAll()_ .

Ahora bien, dado que iniciamos la búsqueda en la parte superior y no filtramos los resultados, generará una gran cantidad de resultados. Sin embargo, ejecutémosla:

    PS C:\Users\stephanie> .\enumeration.ps1
    
    Path
    ----
    LDAP://DC1.corp.com/DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Users,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Computers,DC=corp,DC=com
    LDAP://DC1.corp.com/OU=Domain Controllers,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=System,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=LostAndFound,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Infrastructure,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=ForeignSecurityPrincipals,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Program Data,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Microsoft,CN=Program Data,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=NTDS Quotas,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Managed Service Accounts,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=Keys,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=WinsockServices,CN=System,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=RpcServices,CN=System,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=FileLinks,CN=System,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=VolumeTable,CN=FileLinks,CN=System,DC=corp,DC=com
    LDAP://DC1.corp.com/CN=ObjectMoveTable,CN=FileLinks,CN=System,DC=corp,DC=com
    ...
    

> Listado 19 - Cómo usar nuestro script para buscar anuncios

Como se muestra en la salida truncada del Listado 19, el script genera una gran cantidad de texto. De hecho, estamos recibiendo todos los objetos en todo el dominio. Esto al menos demuestra que el script está funcionando como se esperaba.

Filtrar la salida es bastante simple y existen varias formas de hacerlo. Una forma es configurar un filtro que filtre el atributo [_samAccountType_](https://learn.microsoft.com/en-us/windows/win32/adschema/a-samaccounttype) , que es un atributo que se aplica a todos los objetos de usuario, computadora y grupo.

La documentación oficial revela diferentes valores del atributo _samAccountType_ , pero comenzaremos con 0x30000000 (decimal 805306368), que enumerará a todos los usuarios del dominio. Para implementar el filtro en nuestro script, podemos simplemente agregar el filtro al archivo **$dirsearcher.filter** como se muestra a continuación:

    $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName 
    $LDAP = "LDAP://$PDC/$DN"
    
    $direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
    
    $dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
    $dirsearcher.filter="samAccountType=805306368"
    $dirsearcher.FindAll()
    

> Listado 20 - Uso del atributo samAccountType para filtrar cuentas de usuarios normales

Al ejecutar nuestro script se muestran todos los objetos de usuario en el dominio:

    PS C:\Users\stephanie> .\enumeration.ps1
    
    Path                                                         Properties
    ----                                                         ----------
    LDAP://DC1.corp.com/CN=Administrator,CN=Users,DC=corp,DC=com {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=Guest,CN=Users,DC=corp,DC=com         {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=krbtgt,CN=Users,DC=corp,DC=com        {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=dave,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, usnchanged...}
    LDAP://DC1.corp.com/CN=stephanie,CN=Users,DC=corp,DC=com     {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jeff,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jeffadmin,CN=Users,DC=corp,DC=com     {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=iis_service,CN=Users,DC=corp,DC=com   {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=pete,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jen,CN=Users,DC=corp,DC=com           {logoncount, codepage, objectcategory, dscorepropagatio
    

> Listado 21 - Recepción de todos los usuarios en el dominio filtrando en samAccountType

Es una información muy útil, pero debemos desarrollarla un poco más. Al enumerar AD, nos interesan mucho los _atributos_ de cada objeto, que se almacenan en el campo _Propiedades ._

Sabiendo esto, podemos almacenar los resultados que recibimos de nuestra búsqueda en una nueva variable. Iteraremos a través de cada objeto e imprimiremos cada propiedad en su propia línea mediante un bucle anidado como se muestra a continuación.

    $domainObj = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
    $PDC = $domainObj.PdcRoleOwner.Name
    $DN = ([adsi]'').distinguishedName 
    $LDAP = "LDAP://$PDC/$DN"
    
    $direntry = New-Object System.DirectoryServices.DirectoryEntry($LDAP)
    
    $dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
    $dirsearcher.filter="samAccountType=805306368"
    $result = $dirsearcher.FindAll()
    
    Foreach($obj in $result)
    {
        Foreach($prop in $obj.Properties)
        {
            $prop
        }
    
        Write-Host "-------------------------------"
    }
    

> Listado 22 - Agregar un bucle anidado que imprimirá cada propiedad en su propia línea

Este script completo buscará en AD y filtrará los resultados en función del _tipo de cuenta_ que elijamos, luego colocará los resultados en la nueva variable _$result_ . Luego, filtrará aún más los resultados en función de dos bucles _foreach_ . El primer bucle extraerá los objetos almacenados en _$result_ y los colocará en la variable _$obj_ . El segundo bucle extraerá todas las propiedades de cada objeto y almacenará la información en la variable _$prop_ . Luego, el script imprimirá _$prop_ y presentará el resultado en la terminal.

Si bien el comando _Write-Host_ no es necesario para que el script funcione, sí imprime una línea entre cada objeto. Esto ayuda a que la salida sea un poco más fácil de leer.

El script mostrará una gran cantidad de información, lo que puede resultar abrumador según la cantidad de usuarios del dominio existentes. La lista a continuación muestra una vista parcial de los atributos de _jeffadmin :_

    PS C:\Users\stephanie> .\enumeration.ps1
    ...
    logoncount                     {173}
    codepage                       {0}
    objectcategory                 {CN=Person,CN=Schema,CN=Configuration,DC=corp,DC=com}
    dscorepropagationdata          {9/3/2022 6:25:58 AM, 9/2/2022 11:26:49 PM, 1/1/1601 12:00:00 AM}
    usnchanged                     {52775}
    instancetype                   {4}
    name                           {jeffadmin}
    badpasswordtime                {133086594569025897}
    pwdlastset                     {133066348088894042}
    objectclass                    {top, person, organizationalPerson, user}
    badpwdcount                    {0}
    samaccounttype                 {805306368}
    lastlogontimestamp             {133080434621989766}
    usncreated                     {12821}
    objectguid                     {14 171 173 158 0 247 44 76 161 53 112 209 139 172 33 163}
    memberof                       {CN=Domain Admins,CN=Users,DC=corp,DC=com, CN=Administrators,CN=Builtin,DC=corp,DC=com}
    whencreated                    {9/2/2022 11:26:48 PM}
    adspath                        {LDAP://DC1.corp.com/CN=jeffadmin,CN=Users,DC=corp,DC=com}
    useraccountcontrol             {66048}
    cn                             {jeffadmin}
    countrycode                    {0}
    primarygroupid                 {513}
    whenchanged                    {9/19/2022 6:44:22 AM}
    lockouttime                    {0}
    lastlogon                      {133088312288347545}
    distinguishedname              {CN=jeffadmin,CN=Users,DC=corp,DC=com}
    admincount                     {1}
    samaccountname                 {jeffadmin}
    objectsid                      {1 5 0 0 0 0 0 5 21 0 0 0 30 221 116 118 49 27 70 39 209 101 53 106 82 4 0 0}
    lastlogoff                     {0}
    accountexpires                 {9223372036854775807}
    ...
    

> Listado 23 - Ejecución de secuencia de comandos, impresión de cada atributo para "jeffadmin"

Podemos filtrar en función de cualquier propiedad de cualquier tipo de objeto. En el ejemplo siguiente, hemos realizado dos cambios. Primero, hemos cambiado el filtro para utilizar la propiedad _name_ para mostrar solo información de _jeffadmin_ . Además, hemos agregado _.memberof_ a la variable _$prop_ para mostrar solo los grupos de los que _jeffadmin_ es miembro:

    $dirsearcher = New-Object System.DirectoryServices.DirectorySearcher($direntry)
    $dirsearcher.filter="name=jeffadmin"
    $result = $dirsearcher.FindAll()
    
    Foreach($obj in $result)
    {
        Foreach($prop in $obj.Properties)
        {
            $prop.memberof
        }
    
        Write-Host "-------------------------------"
    }
    

> Listado 24 - Agregar la propiedad de nombre al filtro y solo imprimir el atributo "memberof" en el bucle anidado

Ejecutemos el script:

    PS C:\Users\stephanie> .\enumeration.ps1
    CN=Domain Admins,CN=Users,DC=corp,DC=com
    CN=Administrators,CN=Builtin,DC=corp,DC=com
    

> Listado 25 - Ejecutar script para mostrar únicamente a jeffadmin y de qué grupos es miembro

Esto confirma que _jeffadmin_ es de hecho miembro del grupo _de administradores de dominio_ .

Podemos usar este script para enumerar cualquier objeto disponible en AD. Sin embargo, en el estado actual, esto requeriría que hagamos más modificaciones al script en función de lo que deseamos enumerar.

En cambio, podemos hacer que el script sea más flexible, lo que nos permitirá agregar los parámetros necesarios a través de la línea de comandos. Por ejemplo, podríamos hacer que el script acepte el _tipo de cuenta sam_ que deseamos enumerar como argumento de la línea de comandos.

Hay muchas formas de lograr esto. Una forma es simplemente encapsular la funcionalidad actual del script en una función real. A continuación se muestra un ejemplo de esto.

    function LDAPSearch {
        param (
            [string]$LDAPQuery
        )
    
        $PDC = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain().PdcRoleOwner.Name
        $DistinguishedName = ([adsi]'').distinguishedName
    
        $DirectoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$PDC/$DistinguishedName")
    
        $DirectorySearcher = New-Object System.DirectoryServices.DirectorySearcher($DirectoryEntry, $LDAPQuery)
    
        return $DirectorySearcher.FindAll()
    
    }
    

> Listado 26 - Una función que acepta la entrada del usuario

En la parte superior, declaramos la función con el nombre que elijamos, en este caso _LDAPSearch_ . A continuación, obtiene dinámicamente la cadena de conexión de la ruta LDAP requerida y la agrega a la variable _$DirectoryEntry_ .

Luego, se introduce _DirectoryEntry_ y nuestro parámetro _$LDAPQuery en_ _DirectorySearcher_ . Finalmente, se ejecuta la búsqueda y el resultado se agrega a una matriz, que se muestra en nuestra terminal según nuestras necesidades.

Para utilizar la función, importémosla a la memoria:

    PS C:\Users\stephanie> Import-Module .\function.ps1
    

> Listado 27 – Importando nuestra función a la memoria

Dentro de PowerShell, ahora podemos usar el comando **LDAPSearch** (nuestro nombre de función declarado) para obtener información de AD. Para repetir partes de la enumeración de usuarios que hicimos antes, podemos volver a filtrar por el _samAccountType_ específico :

    PS C:\Users\stephanie> LDAPSearch -LDAPQuery "(samAccountType=805306368)"
    
    Path                                                         Properties
    ----                                                         ----------
    LDAP://DC1.corp.com/CN=Administrator,CN=Users,DC=corp,DC=com {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=Guest,CN=Users,DC=corp,DC=com         {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=krbtgt,CN=Users,DC=corp,DC=com        {logoncount, codepage, objectcategory, description...}
    LDAP://DC1.corp.com/CN=dave,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, usnchanged...}
    LDAP://DC1.corp.com/CN=stephanie,CN=Users,DC=corp,DC=com     {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jeff,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jeffadmin,CN=Users,DC=corp,DC=com     {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=iis_service,CN=Users,DC=corp,DC=com   {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=pete,CN=Users,DC=corp,DC=com          {logoncount, codepage, objectcategory, dscorepropagatio...
    LDAP://DC1.corp.com/CN=jen,CN=Users,DC=corp,DC=com           {logoncount, codepage, objectcategory, dscorepropagatio
    

> Listado 28 - Realizar una búsqueda de usuario utilizando la nueva función

También podemos buscar directamente una _clase de objeto_ , que es un componente de AD que define el tipo de objeto. En este caso, utilizaremos **objectClass=group** para enumerar todos los grupos del dominio:

    PS C:\Users\stephanie> LDAPSearch -LDAPQuery "(objectclass=group)"
    
    ...                                                                                 ----------
    LDAP://DC1.corp.com/CN=Read-only Domain Controllers,CN=Users,DC=corp,DC=com            {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=Enterprise Read-only Domain Controllers,CN=Users,DC=corp,DC=com {iscriticalsystemobject, usnchanged, distinguishedname, grouptype...}
    LDAP://DC1.corp.com/CN=Cloneable Domain Controllers,CN=Users,DC=corp,DC=com            {iscriticalsystemobject, usnchanged, distinguishedname, grouptype...}
    LDAP://DC1.corp.com/CN=Protected Users,CN=Users,DC=corp,DC=com                         {iscriticalsystemobject, usnchanged, distinguishedname, grouptype...}
    LDAP://DC1.corp.com/CN=Key Admins,CN=Users,DC=corp,DC=com                              {iscriticalsystemobject, usnchanged, distinguishedname, grouptype...}
    LDAP://DC1.corp.com/CN=Enterprise Key Admins,CN=Users,DC=corp,DC=com                   {iscriticalsystemobject, usnchanged, distinguishedname, grouptype...}
    LDAP://DC1.corp.com/CN=DnsAdmins,CN=Users,DC=corp,DC=com                               {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=DnsUpdateProxy,CN=Users,DC=corp,DC=com                          {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=Sales Department,DC=corp,DC=com                                 {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=Management Department,DC=corp,DC=com                            {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=Development Department,DC=corp,DC=com                           {usnchanged, distinguishedname, grouptype, whencreated...}
    LDAP://DC1.corp.com/CN=Debug,CN=Users,DC=corp,DC=com                                   {usnchanged, distinguishedname, grouptype, whencreated...}
    

> Listado 29 - Búsqueda de todos los grupos posibles en AD

Nuestro script enumera más grupos que net.exe, incluidos _los operadores de impresión_ , _IIS\_IUSRS_ y otros. Esto se debe a que enumera todos los objetos de AD, incluidos los grupos _locales de dominio_ (no solo los grupos globales).

Para imprimir propiedades y atributos de objetos, necesitaremos implementar los bucles que analizamos anteriormente. Por ahora, haremos esto directamente desde el comando de PowerShell.

Para enumerar todos los grupos disponibles en el dominio y también mostrar los miembros de los usuarios, podemos canalizar la salida a una nueva variable y utilizar un bucle _foreach_ que imprimirá cada propiedad de un grupo. Esto nos permite seleccionar atributos específicos que nos interesen. Por ejemplo, centrémonos en los atributos _CN_ y _de miembro :_

    PS C:\Users\stephanie\Desktop> foreach ($group in $(LDAPSearch -LDAPQuery "(objectCategory=group)")) {
    >> $group.properties | select {$_.cn}, {$_.member}
    >> }
    

> Listado 30 - Uso de "foreach" para iterar a través de los objetos en la variable $group

Aunque este entorno es bastante pequeño, aún así obtuvimos una gran cantidad de resultados. Centrémonos en los tres grupos que notamos antes en nuestra enumeración con net.exe:

    ...
    Sales Department              {CN=Development Department,DC=corp,DC=com, CN=pete,CN=Users,DC=corp,DC=com, CN=stephanie,CN=Users,DC=corp,DC=com}
    Management Department         CN=jen,CN=Users,DC=corp,DC=com
    Development Department        {CN=Management Department,DC=corp,DC=com, CN=pete,CN=Users,DC=corp,DC=com, CN=dave,CN=Users,DC=corp,DC=com}
    ...
    

> Listado 31 - Resultado parcial de nuestra búsqueda anterior

De acuerdo con nuestra búsqueda, hemos ampliado las propiedades de cada objeto, en este caso los objetos _de grupo_ , e imprimimos el atributo _miembro_ para cada grupo.

El listado 31 revela algo inesperado. Antes, cuando enumeramos el grupo _Sales Department_ con net.exe, solo encontramos dos usuarios en él: _pete_ y _stephanie_ . Sin embargo, en este caso, parece que _Development Department_ también es miembro.

Dado que el resultado puede ser algo difícil de leer, busquemos nuevamente los grupos, pero esta vez especifiquemos el _Departamento de Ventas_ en la consulta y canalicémoslo a una variable en nuestra línea de comandos de PowerShell:

    PS C:\Users\stephanie> $sales = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Sales Department))"
    

> Listado 32 – Añadiendo la búsqueda a nuestra variable llamada $sales

Ahora que solo tenemos un objeto en nuestra variable, podemos simplemente imprimir el atributo _miembro_ directamente:

    PS C:\Users\stephanie\Desktop> $sales.properties.member
    CN=Development Department,DC=corp,DC=com
    CN=pete,CN=Users,DC=corp,DC=com
    CN=stephanie,CN=Users,DC=corp,DC=com
    PS C:\Users\stephanie\Desktop>
    

> Listado 33 - Impresión del atributo de miembro en el objeto de grupo Departamento de ventas

El _Departamento de Desarrollo_ es de hecho un miembro del grupo del _Departamento de Ventas_ como se indica en el Listado 33. Esto es algo que pasamos por alto anteriormente con net.exe.

Este es un grupo dentro de un grupo, conocido como _grupo anidado_ . Los grupos anidados son relativamente comunes en AD y se escalan bien, lo que permite flexibilidad y personalización dinámica de la membresía incluso en las implementaciones de AD más grandes.

La herramienta net.exe no detectó esto porque solo enumera objetos _de usuario_ , no objetos de grupo.

Además, net.exe no puede mostrar atributos específicos, lo que resalta el beneficio de las herramientas personalizadas.

Ahora que sabemos que el _Departamento de Desarrollo_ es miembro del _Departamento de Ventas_ , enumerémoslo:

    PS C:\Users\stephanie> $group = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Development Department*))"
    
    PS C:\Users\stephanie> $group.properties.member
    CN=Management Department,DC=corp,DC=com
    CN=pete,CN=Users,DC=corp,DC=com
    CN=dave,CN=Users,DC=corp,DC=com
    

> Listado 34 - Impresión del atributo de miembro en el objeto de grupo del Departamento de Desarrollo

Según el resultado anterior, tenemos otro caso de un grupo anidado, ya que _el Departamento de Gestión_ es miembro del _Departamento de Desarrollo_ . Verifiquemos también este grupo:

    PS C:\Users\stephanie\Desktop> $group = LDAPSearch -LDAPQuery "(&(objectCategory=group)(cn=Management Department*))"
    
    PS C:\Users\stephanie\Desktop> $group.properties.member
    CN=jen,CN=Users,DC=corp,DC=com
    

> Listado 35 - Impresión del atributo de miembro en el objeto de grupo del Departamento de Administración

Finalmente, después de buscar en varios grupos, parece que encontramos el final. Según el resultado del Listado 35, _jen_ es el único miembro del grupo _Departamento de administración_ . Aunque vimos _a jen_ como miembro del grupo _Departamento de administración_ anteriormente en el Listado 31, obtuvimos información adicional sobre las membresías del grupo en este caso enumerando los grupos uno por uno.

Una cosa adicional que se debe tener en cuenta aquí es que, si bien parece que _jen_ es solo una parte del grupo _del Departamento de administración , también es un miembro indirecto de los grupos_ _del Departamento de ventas_ y _del Departamento de desarrollo_ , ya que los grupos generalmente se heredan entre sí. Este es un comportamiento normal en AD; sin embargo, si se configura incorrectamente, los usuarios pueden terminar con más privilegios de los que se pretendía que tuvieran. Esto podría permitir que los atacantes aprovechen la configuración incorrecta para expandir aún más su alcance dentro del dominio comprometido.

Con esto finalizamos el recorrido con nuestro script de PowerShell que invoca clases .NET para ejecutar consultas en AD a través de LDAP. Como hemos comprobado, este enfoque es mucho más potente que ejecutar herramientas como net.exe y ofrece una gran cantidad de opciones de enumeración.

Si bien este script seguramente se puede desarrollar más agregando opciones y funciones adicionales, esto puede requerir más investigación sobre scripts de PowerShell, que está fuera del alcance de este módulo.

Con un conocimiento básico de LDAP y cómo podemos usarlo para comunicarnos con AD mediante PowerShell, en la siguiente sección nos centraremos en un script desarrollado previamente que acelerará nuestro proceso.

Recursos
--------

Algunos de los laboratorios requieren que inicie las máquinas de destino que se indican a continuación.

Tenga en cuenta que las direcciones IP asignadas a sus máquinas de destino pueden no coincidir con las referenciadas en el texto y el video del módulo.

Nombre

(Haga clic para ordenar en orden ascendente)

Dirección IP

Enumeración de Active Directory: cómo agregar funcionalidad de búsqueda a nuestro script (grupo de máquinas virtuales 1)

Iniciar **la enumeración de Active Directory: agregar la funcionalidad de búsqueda a nuestro script: grupo de máquinas virtuales 1** con acceso al navegador Kali

Enumeración de Active Directory: cómo agregar funcionalidad de búsqueda a nuestro script - Grupo de máquinas virtuales 2

Iniciar **la enumeración de Active Directory: agregar la funcionalidad de búsqueda a nuestro script: grupo de máquinas virtuales 2** con acceso al navegador Kali

#### Laboratorios

1.  Inicie el grupo de máquinas virtuales 1 e inicie sesión en CLIENT75 como _stephanie_ . Siga los pasos que se describen en esta sección para agregar la función de búsqueda al script. Encapsule la función del script en una función y repita el proceso de enumeración. ¿Qué clase .NET realiza la búsqueda en Active Directory?

Respuesta

Verificar

2.  Inicie el grupo de máquinas virtuales 2 e inicie sesión en CLIENT75 como _stephanie_ . Utilice el script de PowerShell desarrollado recientemente para enumerar los grupos de dominio, comenzando con _Service Personnel_ . Descifre los grupos anidados y, luego, enumere los atributos del último miembro usuario directo de los grupos anidados para obtener la marca.

Respuesta[Ver sugerencias](#)

Verificar

21.2.4. Enumeración de AD con PowerView
---------------------------------------

Hasta ahora, solo hemos arañado la superficie de la enumeración de Active Directory, centrándonos principalmente en los usuarios y grupos. Si bien las herramientas que hemos utilizado hasta ahora nos han dado un buen comienzo y una comprensión de cómo podemos comunicarnos con AD y obtener información, otros investigadores han creado herramientas más elaboradas para el mismo propósito.

Una opción popular es el script de PowerShell [_PowerView_](https://powersploit.readthedocs.io/en/latest/Recon/) , que incluye muchas funciones para mejorar la efectividad de nuestra enumeración.

A modo de introducción a PowerView, repasemos partes de los pasos de enumeración de la sección anterior. PowerView ya está instalado en la carpeta **C:\\Tools** en CLIENT75. Para usarlo, primero lo importaremos a la memoria:

    PS C:\Tools> Import-Module .\PowerView.ps1
    

> Listado 36 - Importación de PowerView a la memoria

Una vez que hayamos importado PowerView, podremos comenzar a explorar los distintos comandos disponibles. Para obtener una lista de los comandos disponibles en PowerView, consulte la [referencia vinculada](https://powersploit.readthedocs.io/en/latest/Recon/) .

Comencemos ejecutando **Get-NetDomain** , que nos brindará información básica sobre el dominio (para el cual usamos _GetCurrentDomain_ anteriormente):

    PS C:\Tools> Get-NetDomain
    
    Forest                  : corp.com
    DomainControllers       : {DC1.corp.com}
    Children                : {}
    DomainMode              : Unknown
    DomainModeLevel         : 7
    Parent                  :
    PdcRoleOwner            : DC1.corp.com
    RidRoleOwner            : DC1.corp.com
    InfrastructureRoleOwner : DC1.corp.com
    Name                    : corp.com
    

> Listado 37 - Obtención de información de dominio

Al igual que el script que creamos anteriormente, PowerView también utiliza clases .NET para obtener la ruta LDAP requerida y la utiliza para comunicarse con AD.

Ahora obtengamos una lista de todos los usuarios del dominio con **Get-NetUser** :

    PS C:\Tools> Get-NetUser
    
    logoncount             : 113
    iscriticalsystemobject : True
    description            : Built-in account for administering the computer/domain
    distinguishedname      : CN=Administrator,CN=Users,DC=corp,DC=com
    objectclass            : {top, person, organizationalPerson, user}
    lastlogontimestamp     : 9/13/2022 1:03:47 AM
    name                   : Administrator
    objectsid              : S-1-5-21-1987370270-658905905-1781884369-500
    samaccountname         : Administrator
    admincount             : 1
    codepage               : 0
    samaccounttype         : USER_OBJECT
    accountexpires         : NEVER
    cn                     : Administrator
    whenchanged            : 9/13/2022 8:03:47 AM
    instancetype           : 4
    usncreated             : 8196
    objectguid             : e5591000-080d-44c4-89c8-b06574a14d85
    lastlogoff             : 12/31/1600 4:00:00 PM
    objectcategory         : CN=Person,CN=Schema,CN=Configuration,DC=corp,DC=com
    dscorepropagationdata  : {9/2/2022 11:25:58 PM, 9/2/2022 11:25:58 PM, 9/2/2022 11:10:49 PM, 1/1/1601 6:12:16 PM}
    memberof               : {CN=Group Policy Creator Owners,CN=Users,DC=corp,DC=com, CN=Domain Admins,CN=Users,DC=corp,DC=com, CN=Enterprise
                             Admins,CN=Users,DC=corp,DC=com, CN=Schema Admins,CN=Users,DC=corp,DC=com...}
    lastlogon              : 9/14/2022 2:37:15 AM
    ...
    

> Listado 38 - Consulta de usuarios en el dominio

Get-NetUser enumera automáticamente todos los atributos de los objetos de usuario. Esto presenta una gran cantidad de información que puede resultar difícil de digerir.

En el script que creamos anteriormente, usamos bucles para imprimir ciertos atributos en función de la información obtenida. Sin embargo, con PowerView podemos simplemente canalizar la salida a **select** , donde podemos elegir los atributos que nos interesan.

El resultado del listado 38 revela que el atributo _cn_ contiene el nombre de usuario del usuario. Transfiramos el resultado a **select** y elegimos el atributo **cn :**

    PS C:\Tools> Get-NetUser | select cn
    
    cn
    --
    Administrator
    Guest
    krbtgt
    dave
    stephanie
    jeff
    jeffadmin
    iis_service
    pete
    jen
    

> Listado 39 - Consulta de usuarios mediante sentencia select

Esto produjo una lista limpia de usuarios en el dominio.

Al enumerar AD, hay muchos atributos interesantes que buscar. Por ejemplo, si un usuario está inactivo (no ha cambiado su contraseña ni ha iniciado sesión recientemente), provocaremos menos interferencias y llamaremos menos la atención si tomamos el control de esa cuenta durante la interacción. Además, si un usuario no ha cambiado su contraseña desde un cambio reciente en la política de contraseñas, su contraseña puede ser más débil que la política actual. Esto podría hacerla más vulnerable a ataques de contraseñas.

Esto es algo que podemos investigar fácilmente. Ejecutemos **Get-NetUser** nuevamente, esta vez canalizando la salida hacia **select** y extrayendo estos atributos.

    PS C:\Tools> Get-NetUser | select cn,pwdlastset,lastlogon
    
    cn            pwdlastset            lastlogon
    --            ----------            ---------
    Administrator 8/16/2022 5:27:22 PM  9/14/2022 2:37:15 AM
    Guest         12/31/1600 4:00:00 PM 12/31/1600 4:00:00 PM
    krbtgt        9/2/2022 4:10:48 PM   12/31/1600 4:00:00 PM
    dave          9/7/2022 9:54:57 AM   9/14/2022 2:57:28 AM
    stephanie     9/2/2022 4:23:38 PM   12/31/1600 4:00:00 PM
    jeff          9/2/2022 4:27:20 PM   9/14/2022 2:54:55 AM
    jeffadmin     9/2/2022 4:26:48 PM   9/14/2022 2:26:37 AM
    iis_service   9/7/2022 5:38:43 AM   9/14/2022 2:35:55 AM
    pete          9/6/2022 12:41:54 PM  9/13/2022 8:37:09 AM
    jen           9/6/2022 12:43:01 PM  9/13/2022 8:36:55 AM
    

> Listado 40 - Consulta de usuarios que muestran pwdlastset y lastlogon

Como se indica en el Listado 40, tenemos una bonita lista que nos muestra cuándo los usuarios cambiaron su contraseña por última vez, así como cuándo iniciaron sesión en el dominio por última vez.

De manera similar, podemos utilizar **Get-NetGroup** para enumerar grupos:

    PS C:\Tools> Get-NetGroup | select cn
    
    cn
    --
    ...
    Key Admins
    Enterprise Key Admins
    DnsAdmins
    DnsUpdateProxy
    Sales Department
    Management Department
    Development Department
    Debug
    

> Listado 41 - Consulta de grupos en el dominio mediante PowerView

Enumerar grupos específicos con PowerView es fácil. Aunque no realizaremos el proceso de desentrañar grupos anidados en este caso, investiguemos el **Departamento de ventas** mediante **Get-NetGroup** y canalicemos el resultado a **select member** :

    PS C:\Tools> Get-NetGroup "Sales Department" | select member
    
    member
    ------
    {CN=Development Department,DC=corp,DC=com, CN=pete,CN=Users,DC=corp,DC=com, CN=stephanie,CN=Users,DC=corp,DC=com}
    

> Listado 42 - Enumeración del grupo "Departamento de Ventas"

Ahora que esencialmente hemos recreado la funcionalidad de nuestro script anterior, estamos listos para explorar más atributos y técnicas de enumeración.

Recursos
--------

Algunos de los laboratorios requieren que inicie las máquinas de destino que se indican a continuación.

Tenga en cuenta que las direcciones IP asignadas a sus máquinas de destino pueden no coincidir con las referenciadas en el texto y el video del módulo.

Nombre

(Haga clic para ordenar en orden ascendente)

Dirección IP

Enumeración de Active Directory: Enumeración de AD con PowerView: grupo de máquinas virtuales 1

Iniciar **enumeración de Active Directory: enumeración de AD con PowerView: grupo de máquinas virtuales 1** con acceso al navegador Kali

Enumeración de Active Directory: Enumeración de AD con PowerView: grupo de máquinas virtuales 2

Iniciar **enumeración de Active Directory: enumeración de AD con PowerView: grupo de máquinas virtuales 2** con acceso al navegador Kali

#### Laboratorios

1.  Inicie el grupo de máquinas virtuales 1 e inicie sesión en CLIENT75 como _stephanie_ . Importe el script de PowerView a la memoria y repita los pasos de enumeración que se describen en esta sección. ¿Qué comando podemos usar con PowerView para enumerar los grupos de dominio?

Respuesta

Verificar

2.  Inicie el grupo de máquinas virtuales 2 e inicie sesión en CLIENT75 como _stephanie . Utilice PowerView para enumerar el dominio_ _corp.com_ modificado . ¿Qué usuario nuevo forma parte del grupo _de administradores de dominio_ ?

Respuesta[Ver sugerencias](#)

Verificar

3.  Continúe enumerando el dominio _corp.com_ en el grupo de VM 2. Enumere en qué oficina está trabajando el usuario _fred para obtener la bandera._

Respuesta

Verificar

21.3. Enumeración manual: ampliando nuestro repertorio
------------------------------------------------------

Esta unidad de aprendizaje cubre los siguientes objetivos de aprendizaje:

*   Enumerar sistemas operativos
*   Enumerar permisos y usuarios conectados
*   Enumerar a través de nombres principales de servicio
*   Enumerar permisos de objetos
*   Explorar recursos compartidos de dominio

Ahora que estamos familiarizados con LDAP y tenemos algunas herramientas en nuestro conjunto, exploremos más a fondo el dominio.

Nuestro objetivo es utilizar toda esta información para crear un _mapa de dominio_ . Si bien no es necesario que dibujemos un mapa nosotros mismos, es una buena idea intentar visualizar cómo está configurado el dominio y comprender la relación entre los objetos. Visualizar el entorno puede facilitar la búsqueda de posibles vectores de ataque.

21.3.1. Enumerating Operating Systems
-------------------------------------

In a typical penetration test, we use various recon tools in order to detect which operating system a client or server is using. We can, however, enumerate this from Active Directory.

Let's use the **Get-NetComputer** PowerView command to enumerate the computer objects in the domain.

    PS C:\Tools> Get-NetComputer
    
    pwdlastset                    : 10/2/2022 10:19:40 PM
    logoncount                    : 319
    msds-generationid             : {89, 27, 90, 188...}
    serverreferencebl             : CN=DC1,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=corp,DC=com
    badpasswordtime               : 12/31/1600 4:00:00 PM
    distinguishedname             : CN=DC1,OU=Domain Controllers,DC=corp,DC=com
    objectclass                   : {top, person, organizationalPerson, user...}
    lastlogontimestamp            : 10/13/2022 11:37:06 AM
    name                          : DC1
    objectsid                     : S-1-5-21-1987370270-658905905-1781884369-1000
    samaccountname                : DC1$
    localpolicyflags              : 0
    codepage                      : 0
    samaccounttype                : MACHINE_ACCOUNT
    whenchanged                   : 10/13/2022 6:37:06 PM
    accountexpires                : NEVER
    countrycode                   : 0
    operatingsystem               : Windows Server 2022 Standard
    instancetype                  : 4
    msdfsr-computerreferencebl    : CN=DC1,CN=Topology,CN=Domain System Volume,CN=DFSR-GlobalSettings,CN=System,DC=corp,DC=com
    objectguid                    : 8db9e06d-068f-41bc-945d-221622bca952
    operatingsystemversion        : 10.0 (20348)
    lastlogoff                    : 12/31/1600 4:00:00 PM
    objectcategory                : CN=Computer,CN=Schema,CN=Configuration,DC=corp,DC=com
    dscorepropagationdata         : {9/2/2022 11:10:48 PM, 1/1/1601 12:00:01 AM}
    serviceprincipalname          : {TERMSRV/DC1, TERMSRV/DC1.corp.com, Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/DC1.corp.com, ldap/DC1.corp.com/ForestDnsZones.corp.com...}
    usncreated                    : 12293
    lastlogon                     : 10/18/2022 3:37:56 AM
    badpwdcount                   : 0
    cn                            : DC1
    useraccountcontrol            : SERVER_TRUST_ACCOUNT, TRUSTED_FOR_DELEGATION
    whencreated                   : 9/2/2022 11:10:48 PM
    primarygroupid                : 516
    iscriticalsystemobject        : True
    msds-supportedencryptiontypes : 28
    usnchanged                    : 178663
    ridsetreferences              : CN=RID Set,CN=DC1,OU=Domain Controllers,DC=corp,DC=com
    dnshostname                   : DC1.corp.com
    

> Listing 43 - Partial domain computer overview

There are many interesting attributes, but for now we'll search for the operating system and hostnames. Let's pipe the output into **select** and clean up our list.

    PS C:\Tools> Get-NetComputer | select operatingsystem,dnshostname
    
    operatingsystem              dnshostname
    ---------------              -----------
    Windows Server 2022 Standard DC1.corp.com
    Windows Server 2022 Standard web04.corp.com
    Windows Server 2022 Standard FILES04.corp.com
    Windows 11 Pro               client74.corp.com
    Windows 11 Pro               client75.corp.com
    Windows 10 Pro               CLIENT76.corp.com
    

> Listing 44 - Displaying OS and hostname

The output reveals a total of six computers in this domain, three of which are servers, including one DC.

It's a good idea to grab this information early in the assessment to determine the relative age of the systems and to locate potentially weak targets. According to the information we've gathered so far, the machine with the oldest OS appears to be running Windows 10. Additionally, it appears we are dealing with a web server and a file server that will require our attention at some point as well.

So far in our enumeration we have obtained a nice list of all objects in the domain as well as their attributes. In the next section, we will continue using this information to determine the relationships between the various objects in search of potential attack vectors.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Name

(Click to sort ascending)

IP Address

Active Directory Enumeration - Enumerating Operating Systems - VM Group 1

Start **Active Directory Enumeration - Enumerating Operating Systems - VM Group 1** with Kali browser access

Active Directory Enumeration - Enumerating Operating Systems - VM Group 2

Start **Active Directory Enumeration - Enumerating Operating Systems - VM Group 2** with Kali browser access

#### Labs

1.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Repeat the PowerView enumeration steps as outlined in this section. What is the _DistinguishedName_ for the WEB04 machine?

Answer

Verify

2.  Continue enumerating the operating systems in VM Group 1. What is the exact operating system version for _FILES04_? Make sure to provide both the major and minor version number in the answer.

Answer[View hints](#)

Verify

3.  Start VM Group 2 and log in to _CLIENT75_ as _stephanie_. Use PowerView to enumerate the operating systems in the modified _corp.com_ domain to obtain the flag.

Answer

Verify

21.3.2. Obtener una descripción general: permisos y usuarios conectados
-----------------------------------------------------------------------

Ahora que tenemos una lista clara de equipos, usuarios y grupos en el dominio, continuaremos con nuestra enumeración y nos centraremos en las relaciones entre la mayor cantidad posible de objetos. Estas relaciones suelen desempeñar un papel clave durante un ataque y nuestro objetivo es crear un _mapa_ del dominio para encontrar posibles vectores de ataque.

Por ejemplo, cuando un usuario inicia sesión en el dominio, sus credenciales se almacenan en la memoria caché de la computadora desde la que inició sesión.

Si logramos robar esas credenciales, podremos usarlas para autenticarnos como usuario del dominio e incluso podremos aumentar nuestros privilegios de dominio.

Sin embargo, durante una evaluación de AD, es posible que no siempre queramos aumentar nuestros privilegios de inmediato.

En cambio, es importante establecer un buen punto de apoyo y nuestro objetivo, como mínimo, debería ser mantener nuestro acceso. Si podemos comprometer a otros usuarios que tengan los mismos permisos que el usuario al que ya tenemos acceso, esto nos permitirá mantener nuestro punto de apoyo. Si, por ejemplo, se restablece la contraseña del usuario al que obtuvimos acceso originalmente, o los administradores del sistema detectan una actividad sospechosa y desactivan la cuenta, seguiríamos teniendo acceso al dominio a través de otros usuarios a los que comprometimos.

Cuando llega el momento de aumentar nuestros privilegios, no necesariamente tenemos que aumentarlos inmediatamente a _Administradores de dominio_ porque puede haber otras cuentas que tengan privilegios más altos que un usuario de dominio normal, incluso si no son necesariamente parte del grupo de _Administradores de dominio_ . _Las Cuentas de servicio_ , que analizaremos más adelante, son un buen ejemplo de esto. Aunque es posible que no siempre tengan el mayor privilegio posible, pueden tener más permisos que un usuario de dominio normal, como privilegios de administrador local en servidores específicos.

Además, los datos más importantes y confidenciales de una organización pueden almacenarse en ubicaciones que no requieren privilegios de administrador de dominio, como una base de datos o un servidor de archivos. Esto significa que obtener privilegios de administrador de dominio no siempre debe ser el objetivo final durante una evaluación, ya que es posible que podamos acceder a las "joyas de la corona" de una organización a través de otros usuarios del dominio.

Sin embargo, en nuestros Challenge Labs el objetivo es lograr privilegios de administrador de dominio.

Cuando un atacante o un evaluador de penetración mejora el acceso a través de múltiples cuentas de nivel superior para alcanzar un objetivo, se conoce como _compromiso encadenado_ .

Para encontrar posibles rutas de ataque, necesitaremos obtener más información sobre nuestro usuario inicial y ver a qué más tenemos acceso en el dominio. También debemos averiguar dónde están conectados otros usuarios. Profundicemos en eso ahora.

_El comando Find-LocalAdminAccess_ de PowerView escanea la red en un intento de determinar si nuestro usuario actual tiene permisos administrativos en alguna computadora del dominio. El comando se basa en la [_función OpenServiceW_](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-openservicew) , que se conectará al _Administrador de control de servicios_ (SCM) en las máquinas de destino. El SCM básicamente mantiene una base de datos de servicios y controladores instalados en computadoras Windows. PowerView intentará abrir esta base de datos con el derecho de acceso _SC\_MANAGER\_ALL\_ACCESS_ , que requiere privilegios administrativos, y si la conexión es exitosa, PowerView considerará que nuestro usuario actual tiene privilegios administrativos en la máquina de destino.

Ejecutemos **Find-LocalAdminAccess** contra _corp.com_ . Si bien el comando admite parámetros como _Computername_ y _Credentials_ , lo ejecutaremos sin parámetros en este caso, ya que nos interesa enumerar todos los equipos y ya iniciamos sesión como _stephanie_ . En otras palabras, estamos _rociando_ el entorno para encontrar un posible acceso administrativo local en los equipos bajo el contexto de usuario actual.

Dependiendo del tamaño del entorno, es posible que la ejecución de Find-LocalAdminAccess tarde unos minutos en finalizar.

    PS C:\Tools> Find-LocalAdminAccess
    client74.corp.com
    

> Listado 45 - Escaneando el dominio para encontrar privilegios administrativos locales para nuestro usuario

Esto revela que _Stephanie_ tiene privilegios administrativos en CLIENT74. Si bien puede resultar tentador iniciar sesión en CLIENT74 y verificar los permisos de inmediato, esta es una buena oportunidad para hacer una visión más amplia y generalizar.

Las pruebas de penetración pueden llevarnos en muchas direcciones diferentes y, si bien definitivamente debemos seguir los diferentes caminos en función de nuestras interacciones, debemos ceñirnos a nuestro cronograma/plan la mayor parte del tiempo para mantener un enfoque disciplinado.

Continuemos intentando visualizar cómo se conectan entre sí los ordenadores y los usuarios. El primer paso de este proceso será obtener información como qué usuario ha iniciado sesión en qué ordenador.

Históricamente, las dos API de Windows más confiables que podrían (y aún pueden) ayudarnos a lograr estos objetivos son [_NetWkstaUserEnum_](https://learn.microsoft.com/en-us/windows/win32/api/lmwksta/nf-lmwksta-netwkstauserenum) y [_NetSessionEnum_](https://learn.microsoft.com/en-us/windows/win32/api/lmshare/nf-lmshare-netsessionenum) . La primera requiere privilegios administrativos, mientras que la segunda no. Sin embargo, Windows ha experimentado cambios en los últimos años, lo que posiblemente dificulte el descubrimiento de la enumeración de usuarios conectados.

**El comando Get-NetSession** de PowerView utiliza las API _NetWkstaUserEnum_ y _NetSessionEnum_ en segundo plano. Intentemos ejecutarlo en algunas de las máquinas del dominio y veamos si podemos encontrar usuarios conectados:

    PS C:\Tools> Get-NetSession -ComputerName files04
    
    PS C:\Tools> Get-NetSession -ComputerName web04
    PS C:\Tools>
    

> Listado 46 - Comprobación de usuarios conectados con Get-NetSession

Como se indicó anteriormente, no recibimos ningún resultado. Una explicación sencilla sería que no hay usuarios conectados a las máquinas. Sin embargo, para asegurarnos de que no recibimos ningún mensaje de error, agreguemos el indicador **\-Verbose :**

    PS C:\Tools> Get-NetSession -ComputerName files04 -Verbose
    VERBOSE: [Get-NetSession] Error: Access is denied
    
    PS C:\Tools> Get-NetSession -ComputerName web04 -Verbose
    VERBOSE: [Get-NetSession] Error: Access is denied
    

> Listado 47 - Añadiendo verbosidad a nuestro comando Get-NetSession

Lamentablemente, parece que _NetSessionEnum_ no funciona en este caso y devuelve un mensaje de error que indica que se ha denegado el acceso. Lo más probable es que esto signifique que no se nos permite ejecutar la consulta y, según el mensaje de error, puede tener algo que ver con los privilegios.

Dado que podemos tener privilegios administrativos en CLIENT74 con _stephanie_ , ejecutemos **Get-NetSession** contra esa máquina e inspeccionemos la salida allí también:

    PS C:\Tools> Get-NetSession -ComputerName client74
    
    CName        : \\192.168.50.75
    UserName     : stephanie
    Time         : 8
    IdleTime     : 0
    ComputerName : client74
    

> Listado 48 - Ejecución de Get-NetSession en CLIENT74

Esta vez recibimos más información. Sin embargo, si observamos más de cerca el resultado, la dirección IP en _CName_ (192.168.50.75) no coincide con la dirección IP de CLIENT74. De hecho, coincide con la dirección IP de nuestra máquina actual, que es CLIENT75. Como no hemos generado ninguna sesión en CLIENT74, algo parece estar mal también en este caso.

En una interacción en el mundo real, o incluso en los laboratorios de desafíos, podríamos aceptar que enumerar sesiones con PowerView no funciona e intentar usar una herramienta diferente. Sin embargo, usemos esto como una oportunidad de aprendizaje y analicemos en profundidad la API _NetSessionEnum_ e intentemos averiguar exactamente por qué no funciona en nuestro caso.

Según la documentación de [_NetSessionEnum_](https://learn.microsoft.com/en-us/windows/win32/api/lmshare/nf-lmshare-netsessionenum) , existen cinco niveles de consulta posibles: 0,1,2,10,502. El nivel 0 solo devuelve el nombre de la computadora que establece la sesión. Los niveles 1 y 2 devuelven más información, pero requieren privilegios administrativos.

Esto nos deja con los niveles 10 y 502. Ambos deberían devolver información como el nombre de la computadora y el nombre del usuario que establece la conexión. De manera predeterminada, PowerView utiliza el nivel de consulta 10 con _NetSessionEnum_ , que debería brindarnos la información que nos interesa.

Los permisos necesarios para enumerar sesiones con _NetSessionEnum_ se definen en la clave de registro **SrvsvcSessionInfo** , que se encuentra en la sección **HKEY\_LOCAL\_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\LanmanServer\\DefaultSecurity** .

Usaremos la máquina con Windows 11 en la que estamos conectados actualmente para verificar los permisos. Si bien puede tener permisos diferentes a los de las otras máquinas del entorno, puede darnos una idea de lo que está sucediendo.

Para ver los permisos, utilizaremos el cmdlet [**Get-Acl**](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-acl?view=powershell-7.3) de PowerShell . Este comando recuperará básicamente los permisos del objeto que definimos con el indicador **\-Path** y los imprimirá en nuestro indicador de PowerShell.

    PS C:\Tools> Get-Acl -Path HKLM:SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\ | fl
    
    Path   : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\LanmanServer\DefaultSecurity\
    Owner  : NT AUTHORITY\SYSTEM
    Group  : NT AUTHORITY\SYSTEM
    Access : BUILTIN\Users Allow  ReadKey
             BUILTIN\Administrators Allow  FullControl
             NT AUTHORITY\SYSTEM Allow  FullControl
             CREATOR OWNER Allow  FullControl
             APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES Allow  ReadKey
             S-1-15-3-1024-1065365936-1281604716-3511738428-1654721687-432734479-3232135806-4053264122-3456934681 Allow  ReadKey
    

> Listado 49 - Visualización de permisos en el subárbol de registro DefaultSecurity

La salida resaltada en el Listado 49 revela los grupos y usuarios que tienen _FullControl_ o _ReadKey_ , lo que significa que todos pueden leer la clave **SrvsvcSessionInfo .**

Sin embargo, el grupo _BUILTIN_ , el grupo _NT AUTHORITY_ , _CREATOR OWNER_ y _APPLICATION PACKAGE AUTHORITY_ están definidos por el sistema y no permiten que _NetSessionEnum_ enumere esta clave de registro desde un punto de vista remoto.

La cadena larga que aparece al final de la salida es, según la [documentación de Microsoft](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/sids-not-resolve-into-friendly-names) , un _SID de capacidad_ . De hecho, la documentación hace referencia al SID exacto que aparece en nuestra salida.

Un SID de capacidad es un símbolo de autoridad _infalsificable_ que otorga a un componente de Windows o a una aplicación universal de Windows acceso a varios recursos. Sin embargo, no nos dará acceso remoto a la clave de registro de interés.

In older Windows versions (which Microsoft does not specify), _Authenticated Users_ were allowed to access the registry hive and obtain information from the **SrvsvcSessionInfo** key. However, following the _least privilege_ principle, regular domain users should not be able to acquire this information within the domain, which is likely part of the reason the permissions for the registry hive changed as well. In this case, due to permissions, we can be certain that _NetSessionEnum_ will not be able to obtain this type of information on default Windows 11.

Now let's get a better sense of the operating system versions in use. We can do this with **Net-GetComputer**, this time including the **operatingsystemversion** attribute:

    PS C:\Tools> Get-NetComputer | select dnshostname,operatingsystem,operatingsystemversion
    
    dnshostname       operatingsystem              operatingsystemversion
    -----------       ---------------              ----------------------
    DC1.corp.com      Windows Server 2022 Standard 10.0 (20348)
    web04.corp.com    Windows Server 2022 Standard 10.0 (20348)
    FILES04.corp.com  Windows Server 2022 Standard 10.0 (20348)
    client74.corp.com Windows 11 Pro               10.0 (22000)
    client75.corp.com Windows 11 Pro               10.0 (22000)
    CLIENT76.corp.com Windows 10 Pro               10.0 (16299)
    

> Listing 50 - Querying operating system and version

As we discovered earlier, Windows 10 is the oldest operating system in the environment, and based on the output above, it runs version 16299, otherwise known as [build 1709](https://learn.microsoft.com/en-us/windows/uwp/whats-new/windows-10-build-16299).

While the documentation from Microsoft is not clear when they made a change to the **HKEY\_LOCAL\_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\LanmanServer\\DefaultSecurity** registry hive, it appears to be around the release of this exact build. It also seems to affect all Windows Server operating systems since Windows Server 2019 build 1809. This creates an issue for us since we won't be able to use PowerView to build the domain map we had in mind.

Even though _NetSessionEnum_ does not work in this case, we should still keep it in our toolkit since it's not uncommon to find older systems in real-world environments.

Fortunately there are other tools we can use, such as the [_PsLoggedOn_](https://learn.microsoft.com/en-us/sysinternals/downloads/psloggedon) application from the [_SysInternals Suite_](https://learn.microsoft.com/en-us/sysinternals/). The documentation states that PsLoggedOn will enumerate the registry keys under **HKEY\_USERS** to retrieve the _security identifiers_ (SID) of logged-in users and convert the SIDs to usernames. PsLoggedOn will also use the _NetSessionEnum_ API to see who is logged on to the computer via resource shares.

One limitation, however, is that PsLoggedOn relies on the _Remote Registry_ service in order to scan the associated key. The Remote Registry service has not been enabled by default on Windows workstations since Windows 8, but system administrators may enable it for various administrative tasks, for backwards compatibility, or for installing monitoring/deployment tools, scripts, agents, etc.

It is also enabled by default on later Windows Server Operating Systems such as Server 2012 R2, 2016 (1607), 2019 (1809), and Server 2022 (21H2). If it is enabled, the service will stop after ten minutes of inactivity to save resources, but it will re-enable (with an _automatic trigger_) once we connect with PsLoggedOn.

With the theory out of the way for now, let's try to run PsLoggedOn against the computers we attempted to enumerate earlier, starting with FILES04 and WEB04. PsLoggedOn is located in **C:\\Tools\\PSTools** on CLIENT75. To use it, we'll simply run it with the target hostname:

    PS C:\Tools\PSTools> .\PsLoggedon.exe \\files04
    
    PsLoggedon v1.35 - See who's logged on
    Copyright (C) 2000-2016 Mark Russinovich
    Sysinternals - www.sysinternals.com
    
    Users logged on locally:
         <unknown time>             CORP\jeff
    Unable to query resource logons
    

> Listing 51 - Using PsLoggedOn to see user logons at Files04

In this case, we discover that _jeff_ is logged in on FILES04 with his domain user account. This is great information, which suggests another potential attack vector. We'll make a note in our documentation.

Let's go ahead and enumerate WEB04 as well:

    PS C:\Tools\PSTools> .\PsLoggedon.exe \\web04
    
    PsLoggedon v1.35 - See who's logged on
    Copyright (C) 2000-2016 Mark Russinovich
    Sysinternals - www.sysinternals.com
    
    No one is logged on locally.
    Unable to query resource logons
    

> Listing 52 - Using PsLoggedOn to see user logons at Web04

According to the output, there are no users logged in on WEB04. This may be a false positive since we cannot know for sure that the Remote Registry service is running, but we didn't receive any error messages, which suggests the output is accurate. For now, we will simply have to trust our enumeration and accept that no users are logged in on the specific server.

As we discovered earlier in this section, it appears that we have administrative privileges on CLIENT74 via _stephanie_, so this is a machine of high interest, and we should enumerate possible sessions there as well. Let's do that now. For educational purposes, we have enabled the Remote Registry service on CLIENT74.

    PS C:\Tools\PSTools> .\PsLoggedon.exe \\client74
    
    PsLoggedon v1.35 - See who's logged on
    Copyright (C) 2000-2016 Mark Russinovich
    Sysinternals - www.sysinternals.com
    
    Users logged on locally:
         <unknown time>             CORP\jeffadmin
    
    Users logged on via resource shares:
         10/5/2022 1:33:32 AM       CORP\stephanie
    

> Listing 53 - Using PsLoggedOn to see user logons at CLIENT74

It appears _jeffadmin_ has an open session on CLIENT74, and the output reveals some very interesting pieces of information. If our enumeration is accurate and we in fact have administrative privileges on CLIENT74, we should be able to log in there and possibly steal _jeffadmin_'s credentials! It would be very tempting to try this immediately, but it's best practice to stay the course and continue our enumeration. After all, our goal is not to get a quick win, but rather to provide a thorough analysis.

Another interesting thing to note in the output is that _stephanie_ is logged on via resource shares. This is shown because PsLoggedOn also uses the _NetSessionEnum_ API, which in this case requires a logon in order to work. This may also explain why we saw a logon earlier for _stephanie_ while using PowerView.

This concludes the enumeration of our compromised user, including the enumeration of active sessions within the domain. Based on the information we have gathered, we have a very interesting attack path that may lead us all the way to domain administrator if pursued.

In the next section, we will continue our enumeration by focusing on a different type of user, more specifically _Service Accounts_.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Name

(Click to sort ascending)

IP Address

Active Directory Enumeration - Getting an Overview - VM Group 1

Start **Active Directory Enumeration - Getting an Overview - VM Group 1** with Kali browser access

Active Directory Enumeration - Getting an Overview - VM Group 2

Start **Active Directory Enumeration - Getting an Overview - VM Group 2** with Kali browser access

#### Labs

1.  What registry key does _NetSessionEnum_ rely on to discover logged on sessions?

Answer[View hints](#)

Verify

2.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Repeat the enumeration steps outlined in this section to find the logged on sessions. Which service must be enabled on the remote machine to make it possible for PsLoggedOn to enumerate sessions?

Answer[View hints](#)

Verify

3.  Start VM Group 2 and log in to CLIENT75 as _stephanie_. Find out which new machine _stephanie_ has administrative privileges on, then log in to that machine and obtain the flag from the Administrator Desktop.

Answer[View hints](#)

Verify

21.3.3. Enumeración mediante nombres principales de servicio
------------------------------------------------------------

So far, we have obtained quite a bit of information, and we are starting to see how things are connected together within the domain. To wrap up our discussion of user enumeration, we'll shift our focus to [_Service Accounts_](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/service-accounts-on-premises), which may also be members of high-privileged groups.

Applications must be executed in the context of an operating system user. If a user launches an application, that user account defines the context. However, services launched by the system itself run in the context of a _Service Account_.

In other words, isolated applications can use a set of predefined service accounts, such as [_LocalSystem_](https://learn.microsoft.com/en-us/windows/win32/services/localsystem-account), [_LocalService_](https://learn.microsoft.com/en-us/windows/win32/services/localservice-account), and [_NetworkService_](https://learn.microsoft.com/en-us/windows/win32/services/networkservice-account).

For more complex applications, a domain user account may be used to provide the needed context while still maintaining access to resources inside the domain.

When applications like [_Exchange_](https://en.wikipedia.org/wiki/Microsoft_Exchange_Server), MS SQL, or _Internet Information Services_ (IIS) are integrated into AD, a unique service instance identifier known as [_Service Principal Name_](https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names) (SPN) associates a service to a specific service account in Active Directory.

[Managed Service Accounts](https://learn.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview), introduced with Windows Server 2008 R2, were designed for complex applications, which require tighter integration with Active Directory.

Larger applications like MS SQL and Microsoft Exchange often required server redundancy when running to guarantee availability, but Managed Service Accounts did not support this. To remedy this, Group Managed Service Accounts were introduced with Windows Server 2012, but this requires that domain controllers run Windows Server 2012 or higher. Because of this, some organizations may still rely on basic Service Accounts.

We can obtain the IP address and port number of applications running on servers integrated with AD by simply enumerating all SPNs in the domain, meaning we don't need to run a broad port scan.

Since the information is registered and stored in AD, it is present on the domain controller. To obtain the data, we will again query the DC, this time searching for specific SPNs.

To enumerate SPNs in the domain, we have multiple options. In this case, we'll use **setspn.exe**, which is installed on Windows by default. We'll use **\-L** to run against both servers and clients in the domain.

While we could iterate through the list of domain users, we previously discovered the _iis\_service_ user. Let's start with that one:

    c:\Tools>setspn -L iis_service
    Registered ServicePrincipalNames for CN=iis_service,CN=Users,DC=corp,DC=com:
            HTTP/web04.corp.com
            HTTP/web04
            HTTP/web04.corp.com:80
    

> Listing 54 - Listing SPN linked to a certain user account

In the Listing above, an SPN is linked to the _iis\_service_ account.

Another way of enumerating SPNs is to let PowerView enumerate all the accounts in the domain. To obtain a clear list of SPNs, we can pipe the output into **select** and choose the **samaccountname** and **serviceprincipalname** attributes:

    PS C:\Tools> Get-NetUser -SPN | select samaccountname,serviceprincipalname
    
    samaccountname serviceprincipalname
    -------------- --------------------
    krbtgt         kadmin/changepw
    iis_service    {HTTP/web04.corp.com, HTTP/web04, HTTP/web04.corp.com:80}
    

> Listing 55 - Listing the SPN accounts in the domain

While we will explore the _krbtgt_ account in upcoming AD-related Modules, for now, we'll continue to focus on _iis service_. The _serviceprincipalname_ of this account is set to "HTTP/web04.corp.com, HTTP/web04, HTTP/web04.corp.com:80", which is indicative of a web server.

Let's attempt to resolve **web04.corp.com** with **nslookup**:

    PS C:\Tools\> nslookup.exe web04.corp.com
    Server:  UnKnown
    Address:  192.168.50.70
    
    Name:    web04.corp.com
    Address:  192.168.50.72
    

> Listing 56 - Resolving the web04.corp.com name

From the result, it's clear that the hostname resolves to an internal IP address. If we browse this to this IP, we find a website that requires a login:

![Figura 1: Inicio de sesión en Web04](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/b2a4ee29cd973bb20105fc10c63b8cdc-login.png)

Figure 1: Web04 Login

Since these types of accounts are used to run services, we can assume that they have more privileges than regular domain user accounts. For now, we'll simply document that _iis\_service_ has a linked SPN, which will be valuable for us in the upcoming AD-related Modules.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Active Directory Enumeration - Enumeration Through Service Principal Names - VM Group 1

#### Labs

1.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Repeat the enumeration steps outlined in this section to enumerate the Service Account. What is the name of the unique service identifier that is used to associate to a specific service in Active Directory?

Answer

Verify

21.3.4. Enumeración de permisos de objetos
------------------------------------------

In this section, we will enumerate specific permissions that are associated with Active Directory objects. Although the technical details of those permissions are complex and out of scope of this Module, it's important that we discuss the basic principles before we start enumeration.

In short, an object in AD may have a set of permissions applied to it with multiple [_Access Control Entries_](https://learn.microsoft.com/en-us/windows/win32/secauthz/access-control-entries) (ACE). These ACEs make up the [_Access Control List_](https://learn.microsoft.com/en-us/windows/win32/secauthz/access-control-lists) (ACL). Each ACE defines whether access to the specific object is allowed or denied.

As a very basic example, let's say a domain user attempts to access a domain share (which is also an object). The targeted object, in this case the share, will then go through a validation check based on the ACL to determine if the user has permissions to the share. This ACL validation involves two main steps. In an attempt to access the share, the user will send an _access token_, which consists of the user identity and permissions. The target object will then validate the token against the list of permissions (the ACL). If the ACL allows the user to access the share, access is granted. Otherwise the request is denied.

AD includes a wealth of permission types that can be used to configure an [ACE](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.activedirectoryrights?view=netframework-4.7.2). However, from an attacker's standpoint, we are mainly interested in a few key permission types. Here's a list of the most interesting ones along with a description of the permissions they provide:

    GenericAll: Full permissions on object
    GenericWrite: Edit certain attributes on the object
    WriteOwner: Change ownership of the object
    WriteDACL: Edit ACE's applied to object
    AllExtendedRights: Change password, reset password, etc.
    ForceChangePassword: Password change for object
    Self (Self-Membership): Add ourselves to for example a group
    

> Listing 57 - AD permission types

The [Microsoft documentation](https://learn.microsoft.com/en-us/windows/win32/secauthz/access-rights-and-access-masks) lists other permissions and describes each in more detail.

We can use **Get-ObjectAcl** to enumerate ACEs with PowerView. To get started, let's enumerate our own user to determine which ACEs are applied to it. We can do this by filtering on **\-Identity**:

    PS C:\Tools> Get-ObjectAcl -Identity stephanie
    
    ...
    ObjectDN               : CN=stephanie,CN=Users,DC=corp,DC=com
    ObjectSID              : S-1-5-21-1987370270-658905905-1781884369-1104
    ActiveDirectoryRights  : ReadProperty
    ObjectAceFlags         : ObjectAceTypePresent
    ObjectAceType          : 4c164200-20c0-11d0-a768-00aa006e0529
    InheritedObjectAceType : 00000000-0000-0000-0000-000000000000
    BinaryLength           : 56
    AceQualifier           : AccessAllowed
    IsCallback             : False
    OpaqueLength           : 0
    AccessMask             : 16
    SecurityIdentifier     : S-1-5-21-1987370270-658905905-1781884369-553
    AceType                : AccessAllowedObject
    AceFlags               : None
    IsInherited            : False
    InheritanceFlags       : None
    PropagationFlags       : None
    AuditFlags             : None
    ...
    

> Listing 58 - Running Get-ObjectAcl specifying our user

The amount of output may seem overwhelming since we enumerated every ACE that grants or denies some sort of permission to _stephanie_. While there are many properties that seem potentially useful, we are primarily interested in those highlighted in the truncated output of Listing 58.

The output lists two [_Security Identifiers_](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-identifiers) (SID), unique values that represent an object in AD. The first (located in the highlighted _ObjectSID_ property) contains the value "S-1-5-21-1987370270-658905905-1781884369-1104", which is rather difficult to read. In order to make sense of the SID, we can use PowerView's **Convert-SidToName** command to convert it to an actual domain object name:

    PS C:\Tools> Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-1104
    CORP\stephanie
    

> Listing 59 - Converting the ObjectISD into name

The conversion reveals that the SID in the _ObjectSID_ property belongs to the _stephanie_ user we are currently using. The _ActiveDirectoryRights_ property describes the type of permission applied to the object. In order to find out who has the _ReadProperty_ permission in this case, we need to convert the _SecurityIdentifier_ value.

Let's use PowerView to convert it into a name we can read:

    PS C:\Tools> Convert-SidToName S-1-5-21-1987370270-658905905-1781884369-553
    CORP\RAS and IAS Servers
    

> Listing 60 - Converting the SecurityIdentifier into name

According to PowerView, the SID in the _SecurityIdentifier_ property belongs to a default AD group named [_RAS and IAS Servers_](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#ras-and-ias-servers).

Taking this information together, the _RAS and IAS Servers_ group has _ReadProperty_ access rights to our user. While this is a common configuration in AD and likely won't give us an attack vector, we have used the example to make sense of the information we have obtained.

In short, we are interested in the _ActiveDirectoryRights_ and _SecurityIdentifier_ for each object we enumerate going forward.

The highest access permission we can have on an object is _GenericAll_. Although there are many other interesting ones as discussed previously in this section, we will use GenericAll as an example in this case.

We can continue to use **Get-ObjectAcl** and select only the properties we are interested in, namely _ActiveDirectoryRights_ and _SecurityIdentifier_. While the _ObjectSID_ is nice to have, we don't need it when we are enumerating specific objects in AD since it will only contain the SID for the object we are in fact enumerating.

Although we should enumerate all objects the domain, let's start with the _Management Department_ group for now. We will check if any users have GenericAll permissions.

To generate clean and manageable output, we'll use the PowerShell **\-eq** flag to filter the **ActiveDirectoryRights** property, only displaying the values that equal **GenericAll**. We'll then pipe the results into **select**, only displaying the **SecurityIdentifier** and **ActiveDirectoryRights** properties:

    PS C:\Tools> Get-ObjectAcl -Identity "Management Department" | ? {$_.ActiveDirectoryRights -eq "GenericAll"} | select SecurityIdentifier,ActiveDirectoryRights
    
    SecurityIdentifier                            ActiveDirectoryRights
    ------------------                            ---------------------
    S-1-5-21-1987370270-658905905-1781884369-512             GenericAll
    S-1-5-21-1987370270-658905905-1781884369-1104            GenericAll
    S-1-5-32-548                                             GenericAll
    S-1-5-18                                                 GenericAll
    S-1-5-21-1987370270-658905905-1781884369-519             GenericAll
    

> Listing 61 - Enumerating ACLs for the Management Group

In this case, we have a total of five objects that have the GenericAll permission on the _Management Department_ object. To make sense of this, let's convert all the SIDs into actual names:

    PS C:\Tools> "S-1-5-21-1987370270-658905905-1781884369-512","S-1-5-21-1987370270-658905905-1781884369-1104","S-1-5-32-548","S-1-5-18","S-1-5-21-1987370270-658905905-1781884369-519" | Convert-SidToName
    CORP\Domain Admins
    CORP\stephanie
    BUILTIN\Account Operators
    Local System
    CORP\Enterprise Admins
    

> Listing 62 - Converting all SIDs that have GenericAll permission on the Management Group

The first SID belongs to the _Domain Admins_ group and the GenericAll permission comes as no surprise since _Domain Admins_ have the highest privilege possible in the domain. What's interesting, however, is to find _stephanie_ in this list. Typically, a regular domain user should not have GenericAll permissions on other objects in AD, so this may be a misconfiguration.

This finding is significant and indicates that _stephanie_ is a powerful account.

When we enumerated the _Management Group_, we discovered that _jen_ was its only member. As an experiment to show the power of misconfigured object permissions, let's try to use our permissions as _stephanie_ to add ourselves to this group with net.exe.

    PS C:\Tools> net group "Management Department" stephanie /add /domain
    The request will be processed at a domain controller for domain corp.com.
    
    The command completed successfully.
    

> Listing 63 - Using "net.exe" to add ourselves to domain group

Based on the output, we should now be a member of the group. We can verify this with **Get-NetGroup**.

    PS C:\Tools> Get-NetGroup "Management Department" | select member
    
    member
    ------
    {CN=jen,CN=Users,DC=corp,DC=com, CN=stephanie,CN=Users,DC=corp,DC=com}
    

> Listing 64 - Running "Get-NetGroup" to enumerate "Management Department"

This reveals that _jen_ is no longer the sole member of the group and that we have successfully added our _stephanie_ user in there as well.

Now that we have abused the GenericAll permission, let's use it to clean up after ourselves by removing our user from the group:

    PS C:\Tools> net group "Management Department" stephanie /del /domain
    The request will be processed at a domain controller for domain corp.com.
    
    The command completed successfully.
    

> Listing 65 - Using "net.exe" to remove ourselves from domain group

Once again we can use PowerView to verify that _jen_ is the sole member of the group:

    PS C:\Tools> Get-NetGroup "Management Department" | select member
    
    member
    ------
    CN=jen,CN=Users,DC=corp,DC=com
    

> Listing 66 - Running "Get-NetGroup" to verify that our user is removed from domain group

Great, the cleanup was successful.

From a system administrator perspective, managing permissions in Active Directory can be a tough task, especially in complex environments. Weak permissions such as the one we saw here are often the go-to vectors for attackers since it can often help us escalate our privileges within the domain.

In this particular case, we enumerated the _Management Group_ object specifically and leveraged _stephanie's_ GenericAll to add our own user to the group. Although it didn't grant us additional domain privileges, this exercise demonstrated the process of discovering and abusing the vast array of permissions that we can leverage in real-world engagements.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Active Directory Enumeration - Enumerating Object Permissions - VM Group 1

#### Labs

1.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Repeat the enumeration steps outlined in this section to get an understanding for the object permissions. What kind of entries makes up an ACL?

Answer

Verify

2.  What is the most powerful ACL we can have on an object in Active Directory?

Answer

Verify

21.3.5. Enumerating Domain Shares
---------------------------------

To wrap up our manual enumeration discussion, we'll shift our focus to domain shares. Domain shares often contain critical information about the environment, which we can use to our advantage.

We'll use PowerView's **Find-DomainShare** function to find the shares in the domain. We could also add the _\-CheckShareAccess_ flag to display shares only available to us. However, we'll skip this flag for now to return a full list, including shares we may target later. Note that it may take a few moments for PowerView to find the shares and list them.

    PS C:\Tools> Find-DomainShare
    
    Name           Type Remark                 ComputerName
    ----           ---- ------                 ------------
    ADMIN$   2147483648 Remote Admin           DC1.corp.com
    C$       2147483648 Default share          DC1.corp.com
    IPC$     2147483651 Remote IPC             DC1.corp.com
    NETLOGON          0 Logon server share     DC1.corp.com
    SYSVOL            0 Logon server share     DC1.corp.com
    ADMIN$   2147483648 Remote Admin           web04.corp.com
    backup            0                        web04.corp.com
    C$       2147483648 Default share          web04.corp.com
    IPC$     2147483651 Remote IPC             web04.corp.com
    ADMIN$   2147483648 Remote Admin           FILES04.corp.com
    C                 0                        FILES04.corp.com
    C$       2147483648 Default share          FILES04.corp.com
    docshare          0 Documentation purposes FILES04.corp.com
    IPC$     2147483651 Remote IPC             FILES04.corp.com
    Tools             0                        FILES04.corp.com
    Users             0                        FILES04.corp.com
    Windows           0                        FILES04.corp.com
    ADMIN$   2147483648 Remote Admin           client74.corp.com
    C$       2147483648 Default share          client74.corp.com
    IPC$     2147483651 Remote IPC             client74.corp.com
    ADMIN$   2147483648 Remote Admin           client75.corp.com
    C$       2147483648 Default share          client75.corp.com
    IPC$     2147483651 Remote IPC             client75.corp.com
    sharing           0                        client75.corp.com
    

> Listing 67 - Domain Share Query

Listing 67 reveals shares from three different servers and a few clients. Although some of these are default domain shares, we should investigate each of them in search of interesting information.

In this instance, we'll first focus on [**SYSVOL**](https://social.technet.microsoft.com/wiki/contents/articles/24160.active-directory-back-to-basics-sysvol.aspx), as it may include files and folders that reside on the domain controller itself. This particular share is typically used for various domain policies and scripts. By default, the **SYSVOL** folder is mapped to **%SystemRoot%\\SYSVOL\\Sysvol\\domain-name** on the domain controller and every domain user has access to it.

    PS C:\Tools> ls \\dc1.corp.com\sysvol\corp.com\
    
        Directory: \\dc1.corp.com\sysvol\corp.com
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d-----         9/21/2022   1:11 AM                Policies
    d-----          9/2/2022   4:08 PM                scripts
    

> Listing 68 - Listing contents of the SYSVOL share

During an assessment, we should investigate every folder we discover in search of interesting items. For now, let's examine the **Policies** folder:

    PS C:\Tools> ls \\dc1.corp.com\sysvol\corp.com\Policies\
    
        Directory: \\dc1.corp.com\sysvol\corp.com\Policies
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d-----         9/21/2022   1:13 AM                oldpolicy
    d-----          9/2/2022   4:08 PM                {31B2F340-016D-11D2-945F-00C04FB984F9}
    d-----          9/2/2022   4:08 PM                {6AC1786C-016F-11D2-945F-00C04fB984F9}
    

> Listing 69 - Listing contents of the "SYSVOL\\policies share"

All the folders are potentially interesting, but we'll explore **oldpolicy** first. Within it, as shown in Listing 70, we find a file named **old-policy-backup.xml**:

    PS C:\Tools> cat \\dc1.corp.com\sysvol\corp.com\Policies\oldpolicy\old-policy-backup.xml
    <?xml version="1.0" encoding="utf-8"?>
    <Groups   clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
      <User   clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}"
              name="Administrator (built-in)"
              image="2"
              changed="2012-05-03 11:45:20"
              uid="{253F4D90-150A-4EFB-BCC8-6E894A9105F7}">
        <Properties
              action="U"
              newName=""
              fullName="admin"
              description="Change local admin"
              cpassword="+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
              changeLogon="0"
              noChange="0"
              neverExpires="0"
              acctDisabled="0"
              userName="Administrator (built-in)"
              expires="2016-02-10" />
      </User>
    </Groups>
    

> Listing 70 - Checking contents of old-policy-backup.xml file

Due to the naming of the folder and the name of the file itself, it appears that this is an older domain policy file. This is a common artifact on domain shares as system administrators often forget them when implementing new policies. In this particular case, the XML file describes an old policy (helpful for learning more about the current policies) and an encrypted password for the local built-in Administrator account. The encrypted password could be extremely valuable for us.

Historically, system administrators often changed local workstation passwords through [_Group Policy Preferences_](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/dn581922(v=ws.11)) (GPP).

However, even though GPP-stored passwords are encrypted with AES-256, the private key for the encryption has been posted on [_MSDN_](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be?redirectedfrom=MSDN#endNote2). We can use this key to decrypt these encrypted passwords. In this case, we'll use the [**gpp-decrypt**](https://www.kali.org/tools/gpp-decrypt/) ruby script in Kali Linux that decrypts a given GPP encrypted string:

    kali@kali:~$ gpp-decrypt "+bsY0V3d4/KgX3VJdO/vyepPfAN1zMFTiQDApgR92JE"
    P@$$w0rd
    

> Listing 71 - Using gpp-decrypt to decrypt the password

As indicated in Listing 71, we successfully decrypted the password and we will make a note of this in our documentation.

Listing 67 also revealed other shares of potential interest. Let's check out **docshare** on **FILES04.corp.com** (which is not a default share).

    PS C:\Tools> ls \\FILES04\docshare
    
        Directory: \\FILES04\docshare
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    d-----         9/21/2022   2:02 AM                docs
    

> Listing 72 - Listing the contents of docsare

Farther in the folder structure, we find a **do-not-share** folder that contains **start-email.txt**:

    PS C:\Tools> ls \\FILES04\docshare\docs\do-not-share
    
        Directory: \\FILES04\docshare\docs\do-not-share
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a----         9/21/2022   2:02 AM           1142 start-email.txt
    

> Listing 73 - Listing the contents of do-not-share

Although this is a very strange name for a folder that is in fact shared, let's check out the content of the file:

    PS C:\Tools> cat \\FILES04\docshare\docs\do-not-share\start-email.txt
    Hi Jeff,
    
    We are excited to have you on the team here in Corp. As Pete mentioned, we have been without a system administrator
    since Dennis left, and we are very happy to have you on board.
    
    Pete mentioned that you had some issues logging in to your Corp account, so I'm sending this email to you on your personal address.
    
    The username I'm sure you already know, but here you have the brand new auto generated password as well: HenchmanPutridBonbon11
    
    As you may be aware, we are taking security more seriously now after the previous breach, so please change the password at first login.
    
    Best Regards
    Stephanie
    
    ...............
    
    Hey Stephanie,
    
    Thank you for the warm welcome. I heard about the previous breach and that Dennis left the company.
    
    Fortunately he gave me a great deal of documentation to go through, although in paper format. I'm in the
    process of digitalizing the documentation so we can all share the knowledge. For now, you can find it in
    the shared folder on the file server.
    
    Thank you for reminding me to change the password, I will do so at the earliest convenience.
    
    Best regards
    Jeff
    

> Listing 74 - Checking the "start-email.txt" file

According to the text in this file, _jeff_ stored an email with a possible cleartext password: _HenchmanPutridBonbon11_! Although the password may have been changed, we will make a note of it in our documentation. Between this password and the password we discovered earlier, we're building a rough profile of the password policy used for both users and computers in the organization. We could use this to create specific wordlists that we can use for password guessing and brute force, if needed.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Active Directory Enumeration - Enumerating Domain Shares - VM Group 1

Active Directory Enumeration - Enumerating Domain Shares - VM Group 2

#### Labs

1.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Repeat the enumeration steps outlined in this section and view the information in the accessible shares. What is the hostname for the server sharing the **SYSVOL** folder in the _corp.com_ domain?

Answer

Verify

2.  Start VM Group 2 and log in to CLIENT75 as _stephanie_. Use PowerView to locate the shares in the modified _corp.com_ domain and enumerate them to obtain the flag.

Answer

Verify

21.4. Active Directory - Automated Enumeration
----------------------------------------------

This Learning Unit covers the following Learning Objectives:

*   Collect domain data using SharpHound
*   Analyze the data using BloodHound

As we've seen so far in this Module, our manual enumeration can be relatively time consuming and can generate a wealth of information that can be difficult to organize.

Although it is important to understand the concepts of manual enumeration, we can also leverage automated tools to speed up the enumeration process and quickly reveal possible attack paths, especially in large environments. Manual and automated tools each have their merits, and most professionals leverage a combination of the two in real-world engagements.

Some automated tools, like [_PingCastle_](https://www.pingcastle.com/),generate gorgeous reports although most require paid licenses for commercial use. In our case, we will focus on [_BloodHound_](https://support.bloodhoundenterprise.io/hc/en-us), an excellent free tool that's extremely useful for analyzing AD environments.

It's worth noting that automated tools generate a great deal of network traffic and many administrators will likely recognize a spike in traffic as we run these tools.

21.4.1. Collecting Data with SharpHound
---------------------------------------

We'll use BloodHound in the next section to analyze, organize and present the data, and the companion data collection tool, [_SharpHound_](https://support.bloodhoundenterprise.io/hc/en-us/articles/17481151861019-SharpHound-Community-Edition) to collect the data. SharpHound is written in C# and uses Windows API functions and LDAP namespace functions similar to those we used manually in the previous sections. For example, SharpHound will attempt to use [_NetWkstaUserEnum_](https://learn.microsoft.com/en-us/windows/win32/api/lmwksta/nf-lmwksta-netwkstauserenum) and [_NetSessionEnum_](https://learn.microsoft.com/en-us/windows/win32/api/lmshare/nf-lmshare-netsessionenum) to enumerate logged-on sessions, just as we did earlier. It will also run queries against the Remote Registry service, which we also leveraged earlier.

It's often best to combine automatic and manual enumeration techniques when assessing Active Directory. Even though we could theoretically gather the same information with a manual approach, graphical relationships often reveal otherwise unnoticed attack paths.

Let's get SharpHound up and running.

SharpHound is available in a few different formats. We can compile it ourselves, use an already compiled executable, or use it as a PowerShell script.

In our case, we will use a PowerShell script. To use the most updated version we will not use the file that is located in **C:\\Tools** on CLIENT75. Instead, we will download the current zip file of [SharpHound](https://github.com/BloodHoundAD/SharpHound/releases) to our Kali machine. Next, we can extract the **Sharphound.ps1** file from the zip file, and transfer it to the CLIENT75 machine. Once we have done that, we can open a PowerShell window and import the script to memory:

    PS C:\Users\stephanie> cd .\Downloads\
    
    PS C:\Users\stephanie\Downloads> powershell -ep bypass
    Windows PowerShell
    Copyright (C) Microsoft Corporation. All rights reserved.
    
    Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows
    PS C:\Users\stephanie\Downloads> Import-Module .\Sharphound.ps1
    

> Listing 75 - Importing the SharpHound script to memory

With SharpHound imported, we can now start collecting domain data. However, in order to run SharpHound, we must first run **Invoke-BloodHound**. This is not intuitive since we're only running SharpHound at this stage. Let's invoke **Get-Help** to learn more about this command.

    PS C:\Users\stephanie\Downloads> Get-Help Invoke-BloodHound
    
    NAME
        Invoke-BloodHound
    
    SYNOPSIS
        Runs the BloodHound C# Ingestor using reflection. The assembly is stored in this file.
    
    
    SYNTAX
        Invoke-BloodHound [-CollectionMethods <String[]>] [-Domain <String>] [-SearchForest] [-Stealth] [-LdapFilter
        <String>] [-DistinguishedName <String>] [-ComputerFile <String>] [-OutputDirectory <String>] [-OutputPrefix
        <String>] [-CacheName <String>] [-MemCache] [-RebuildCache] [-RandomFilenames] [-ZipFilename <String>] [-NoZip]
        [-ZipPassword <String>] [-TrackComputerCalls] [-PrettyPrint] [-LdapUsername <String>] [-LdapPassword <String>]
        [-DomainController <String>] [-LdapPort <Int32>] [-SecureLdap] [-DisableCertVerification] [-DisableSigning]
        [-SkipPortCheck] [-PortCheckTimeout <Int32>] [-SkipPasswordCheck] [-ExcludeDCs] [-Throttle <Int32>] [-Jitter
        <Int32>] [-Threads <Int32>] [-SkipRegistryLoggedOn] [-OverrideUsername <String>] [-RealDNSName <String>]
        [-CollectAllProperties] [-Loop] [-LoopDuration <String>] [-LoopInterval <String>] [-StatusInterval <Int32>]
        [-Verbosity <Int32>] [-Help] [-Version] [<CommonParameters>]
    
    
    DESCRIPTION
        Using reflection and assembly.load, load the compiled BloodHound C# ingestor into memory
        and run it without touching disk. Parameters are converted to the equivalent CLI arguments
        for the SharpHound executable and passed in via reflection. The appropriate function
        calls are made in order to ensure that assembly dependencies are loaded properly.
    
    
    RELATED LINKS
    
    REMARKS
        To see the examples, type: "get-help Invoke-BloodHound -examples".
        For more information, type: "get-help Invoke-BloodHound -detailed".
        For technical information, type: "get-help Invoke-BloodHound -full".
    

> Listing 76 - Checking the SharpHound options

We'll begin with the [**\-CollectionMethod**](https://bloodhound.readthedocs.io/en/latest/data-collection/sharphound-all-flags.html), which describes the various collection methods. In our case, we'll attempt to gather **All** data, which will perform all collection methods except for local group policies.

By default, SharpHound will gather the data in JSON files and automatically zip them for us. This makes it easy for us to transfer the file to Kali Linux later. We'll save this output file on our desktop, with a "corp audit" prefix as shown below:

    PS C:\Users\stephanie\Downloads> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\Users\stephanie\Desktop\ -OutputPrefix "corp audit"
    

> Listing 77 - Running SharpHound to collect domain data

Note that the data collection may take a few moments to finish, depending on the size of the environment we are enumerating. Let's examine SharpHound's output:

    2024-08-10T20:16:00.6554069-07:00|INFORMATION|This version of SharpHound is compatible with the 5.0.0 Release of BloodHound
    2024-08-10T20:16:00.7960323-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, UserRights, CARegistry, DCRegistry, CertServices
    2024-08-10T20:16:00.8429091-07:00|INFORMATION|Initializing SharpHound at 8:16 PM on 8/10/2024
    2024-08-10T20:16:00.8741609-07:00|INFORMATION|Resolved current domain to corp.com
    2024-08-10T20:16:00.9835316-07:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, UserRights, CARegistry, DCRegistry, CertServices
    2024-08-10T20:16:01.0616591-07:00|INFORMATION|Beginning LDAP search for corp.com
    2024-08-10T20:16:01.1241627-07:00|INFORMATION|Beginning LDAP search for corp.com Configuration NC
    2024-08-10T20:16:01.1397817-07:00|INFORMATION|Producer has finished, closing LDAP channel
    2024-08-10T20:16:01.1554037-07:00|INFORMATION|LDAP channel closed, waiting for consumers
    2024-08-10T20:16:01.7022783-07:00|INFORMATION|Consumers finished, closing output channel
    Closing writers
    2024-08-10T20:16:01.7179066-07:00|INFORMATION|Output channel closed, waiting for output task to complete
    2024-08-10T20:16:01.8272851-07:00|INFORMATION|Status: 309 objects finished (+309 Infinity)/s -- Using 118 MB RAM
    2024-08-10T20:16:01.8272851-07:00|INFORMATION|Enumeration finished in 00:00:00.7702863
    2024-08-10T20:16:01.8897888-07:00|INFORMATION|Saving cache with stats: 19 ID to type mappings.
     2 name to SID mappings.
     6 machine sid mappings.
     4 sid to domain mappings.
     0 global catalog mappings.
    2024-08-10T20:16:01.9054062-07:00|INFORMATION|SharpHound Enumeration Completed at 8:16 PM on 8/10/2024! Happy Graphing!
    
    

> Listing 78 - SharpHound output

Based on the output in Listing 78, we scanned a total of 309 objects. This will obviously vary based on how many objects and sessions exist in the domain.

In this case, SharpHound essentially took a snapshot of the domain from the _stephanie_ user, and we should be able to analyze everything the user account has access to. The collected data is stored in the zip file located on our Desktop:

    PS C:\Users\stephanie\Downloads> ls C:\Users\stephanie\Desktop\
    
        Directory: C:\Users\stephanie\Desktop
    
    
    Mode                 LastWriteTime         Length Name
    ----                 -------------         ------ ----
    -a----         8/10/2024   8:16 PM          26255 corp audit_20240810201601_BloodHound.zip
    -a----         8/10/2024   8:16 PM           2110 MTk2MmZkNjItY2IyNC00MWMzLTk5YzMtM2E1ZDcwYThkMzRl.bin
    

> Listing 79 - SharpHound generated files

We'll use this file in the next section as we analyze the data with BloodHound. Sharphound created the **bin** cache file to speed up data collection. This is not needed for our analysis and we can safely delete it.

One thing to note is that SharpHound also supports _looping_, which means that the collector will run cyclical queries of our choosing over a period of time. While the collection method we used above created a _snapshot_ over the domain, running it in a loop may gather additional data as the environment changes. The cache file speeds up the process. For example, if a user logged on after we collected a snapshot, we would have missed it in our analysis. We will not use the looping functionality, but we recommend experimenting with it in the training labs and inspecting the results in BloodHound.

Resources
---------

Some of the labs require you to start the target machine(s) below.

Please note that the IP addresses assigned to your target machines may not match those referenced in the Module text and video.

Active Directory Enumeration - Collecting Data with SharpHound - VM Group 1

#### Labs

1.  Start VM Group 1 and log in to CLIENT75 as _stephanie_. Gather the domain data with SharpHound as outlined in this section. Which function can we use with SharpHound to see changes happening in the domain over a longer period of time?

Answer

Verify

2.  Which syntax in SharpHound allows us to set a password on the resulting .zip file?

Answer

Verify

21.4.2. Analysing Data using BloodHound
---------------------------------------

In this section, we will analyze the domain data using BloodHound in Kali Linux, but it should be noted that we could install the application and required dependencies on Windows-based systems as well.

In order to use BloodHound, we need to start the [_Neo4j_](https://neo4j.com/) service, which is installed by default. Note that when [Bloodhound](https://www.kali.org/tools/bloodhound/) is installed with _APT_, the Neo4j service is automatically installed as well.

Neo4j is essentially an open source [graph database](https://en.wikipedia.org/wiki/Graph_database) (NoSQL) that creates nodes, edges, and properties instead of simple rows and columns. This facilitates the visual representation of our collected data. Let's go ahead and start the Neo4j service:

    kali@kali:~$ sudo neo4j start
    Directories in use:
    home:         /usr/share/neo4j
    config:       /usr/share/neo4j/conf
    logs:         /usr/share/neo4j/logs
    plugins:      /usr/share/neo4j/plugins
    import:       /usr/share/neo4j/import
    data:         /usr/share/neo4j/data
    certificates: /usr/share/neo4j/certificates
    licenses:     /usr/share/neo4j/licenses
    run:          /usr/share/neo4j/run
    Starting Neo4j.
    Started neo4j (pid:334819). It is available at http://localhost:7474
    There may be a short delay until the server is ready.
    

> Listing 80 - Starting the Neo4j service in Kali Linux

As indicated in the output, the Neo4j service is now running and it should be available via the web interface at **http://localhost:7474**. Let's browse this location and authenticate using the default credentials (_neo4j_ as both username and password):

![Figura 2: Primer inicio de sesión en Neo4j](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/3355cd0fa115e245d4865adb149d2151-bhound1.png)

Figure 2: Neo4j First Login

After authenticating with the default credentials, we are prompted for a password change.

![Figura 3: Cambio de contraseña de Neo4j](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/9cf6d952aa5dbd52d63c758b38968b6e-bhound2.png)

Figure 3: Neo4j Password Change

In this case, we can choose any password we'd like; however, we must remember it since we'll also use it to authenticate to the database later.

Once we have changed the password, we can authenticate to the database and run our own queries against it. However, since we haven't imported any data yet there isn't much we can do and we'd rather allow BloodHound to run the queries for us.

With Neo4j running, it's time to start BloodHound as well. We can do this directly from the terminal:

    kali@kali:~$ bloodhound
    

> Listing 81 - Starting BloodHound in Kali Linux

Once we start BloodHound, we are met with an authentication window, asking us to log in to the Neo4j Database:

![Figura 4: Inicio de sesión en BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/a07811682b98af63a48f6edebd44f2d3-bhound3.png)

Figure 4: BloodHound Login

As indicated by the green check mark in the first column, BloodHound has automatically detected that we have the Neo4j database running. In order to log in, we use the _neo4j_ username and the password we created earlier.

Since we haven't imported data yet, we don't have any visual representation of the domain at this point. In order to import the data, we must first transfer the zip file from our Windows machine to our Kali Linux machine. We can then use the _Upload Data_ function on the right side of the GUI to upload the zip file, or drag-and-drop it into BloodHound's main window. Either way, the progress bar indicates the upload progress.

![Figura 5: Carga de datos recopilados](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/8f997331e1f3dcd1fcc2932666ce77b3-bhound4.png)

Figure 5: Uploading Collected Data

Once the upload is finished, we can close the _Upload Progress_ window.

Now it's time to start analyzing the data. Let's first get an idea about how much data the database really contains. To do this, let's click the _Hamburger_ menu at the top-left. This presents the _Database Info_ as shown below:

![Figura 6: Información de la base de datos BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/538fd032517a03cebeb30eb42a0d2016-bhound5.png)

Figure 6: BloodHound DB Info

Our small environment doesn't contain much. But in some cases, especially in a larger environment, the database may take some time to update. In these cases, we can use the _Refresh Database Stats_ button to present an updated view.

Looking at the information, we have discovered five total sessions in the domain, which have been enumerated (using _NetSessionEnum_ and _PsLoggedOn_ techniques we used earlier). Additionally, we have discovered a wealth of ACLs, a total of 10 users, 57 groups, and more.

We'll explain the _Node Info_ later, as there isn't much here at this point. For now, we are mostly interested in the _Analysis_ button. When we click it, we are presented with various pre-built analysis options:

![Figura 7: Descripción general del análisis de BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/2ff0e8c6b63fe3fa227241770807db9f-bhound6.png)

Figure 7: BloodHound Analysis Overview

There are many pre-built analytics queries to experiment with here, and we will not be able to cover all of them in this Module. However, to get started, let's use _Find all Domain Admins_ under _Domain Information_. This presents the graph shown below:

![Figura 8: Administradores de dominio de BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/b4e5987788889d298bee77e09aa44742-bhound7.png)

Figure 8: BloodHound Domain Admins

Each of the circle icons are known as _nodes_, and we can drag them to move them in the interface. In this case the three nodes are connected and BloodHound placed them far apart from each other, so we can simply move them closer to each other to keep everything nice and clean.

In order to see what the two nodes on the left represent, we can hover over them with the mouse pointer, or we can toggle the information by pressing the control button. While toggling on and off the information for each node may be preferred for some analysis, we can also tell BloodHound to show this information by default by clicking _Settings_ (represented as two cogs) on the right side of the interface and setting _Node Label Display_ to _Always Display_:

![Figura 9: Visualización del nodo BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/6436a820a0881112f7f524c31c1c2c2c-bhound8.png)

Figure 9: BloodHound Node Display

Once we make that change, we can close it with the _x_ in the top right corner.

Back on the Bloodhound view, the _Domain Admins_ for the domain are indeed _jeffadmin_ and the _administrator_ account itself. As shown in Figure below, BloodHound shows an edge in the form of a line between the user objects and the _Domain Admins_ group, indicating the relationship, which in this case tells us that the particular users are a member of the given group:

![Figura 10: Visualización del nodo BloodHound2](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/29b23a912d5f8e84f5c0df787dc9d351-bhound9.png)

Figure 10: BloodHound Node Display2

Although BloodHound is capable of deep analysis, much of its functionality is out of scope for this Module. For now, we'll focus on the _Shortest Paths_ shown in the _Analysis_ tab.

One of the strengths of BloodHound is its ability to automatically attempt to find the shortest path possible to reach our goal, whether that goal is to take over a particular computer, user, or group.

Let's go back to the Analysis screen and scroll down to the **Shortest Path** section and select the _Find Shortest Paths to Domain Admins_. This provides a nice overview and doesn't require any parameters.

![Figura 11: Ruta más corta de BloodHound DA](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/66f78221ebcab3baa9f4dcb5600395d7-bhound10.png)

Figure 11: BloodHound Shortest Path DA

This reveals the true power of BloodHound.

Now when we run it, we can analyze this graph to determine our best attack approach. In this case, the graph will reveal a few things we didn't catch in our earlier enumeration.

For example, let's focus on the relationship between _stephanie_ and CLIENT74, which we saw in our earlier enumeration. To get more information, we can hover the mouse over the string that indicates the connection between the node to see what kind of connection it really is:

![Figura 12: Stephanie RDP, la sabuesa de sangre](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/61e1abfbe08f44b8c5893121183e004d-bhound11.png)

Figure 12: BloodHound Stephanie RDP

The small pop-up says _AdminTo_, and this indicates that _stephanie_ indeed has administrative privileges on CLIENT74. If we right-click the line between the nodes and click _? Help_, BloodHound will show additional information:

![Figura 13: Ayuda de BloodHound](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/4ebedc1542c7a3eb4c40b7764f631038-bhound12.png)

Figure 13: BloodHound Help

As indicated in the information above, _stephanie_ has administrative privileges on CLIENT74 and has several ways to obtain code execution on it.

In the _? Help_ menu BloodHound also offers information in the _Abuse_ tab, which will tell us more about the possible attack we can take on the given path. It also contains _Opsec_ information as what to look out for when it comes to being detected, as well as references to the information displayed.

After further reading of Figure 11, and after further inspection of the graph, we discover the connection _jeffadmin_ has to CLIENT74. This means that the credentials for _jeffadmin_ may be cached on the machine, which could be fatal for the organization. If we are able to take advantage of the given attack path and steal the credentials for _jeffadmin_, we should be able to log in as him and become domain administrator through his _Domain Admins_ membership.

This plays directly into the second _Shortest Path_ we'd like to show for this Module, namely the _Shortest Paths to Domain Admins from Owned Principals_. If we run this query against _corp.com_ without configuring BloodHound, we receive a "NO DATA RETURNED FROM QUERY" message.

However, the _Owned Principals_ plays a big role here, and refers to the objects we are currently in control of in the domain. In order to analyze, we can mark any object we'd like as _owned_ in BloodHound, even if we haven't obtained access to them. Sometimes it is a good idea to think in the lines of "what if" when it comes to AD assessments. In this case however, we will leave the imagination on the side and focus on the objects we in fact have control over.

The only object we know for a fact we have control over is the _stephanie_ user, and we have partial control over CLIENT75, since that is where we are logged in. We do not have administrative privileges, so we may need to think about doing privilege escalation on the machine later, but for now, let's say that we have control over it.

In order for us to obtain an _owned principal_ in BloodHound, we will run a search (top left), right click the object that shows in the middle of the screen, and click _Mark User as Owned_. Since we are already at a the user _stephanie_ we can simply right-click and select _Mark User as Owned_ A principal marked as _owned_ is shown in BloodHound with a skull icon next to the node itself.

![Figura 14: Marca BloodHound en propiedad](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/363e76fb6ca0f6c31155015ddfb2a6a6-bhound13.png)

Figure 14: BloodHound Mark Owned

One thing to note here is that if we click the icon for the object we are searching, it will be placed into the _Node Info_ button where we can read more about the object itself.

We'll repeat the process for CLIENT75 as well, however in this case we click _Mark Computer as Owned_, and we end up having two _owned principals_.

:::info It's a good idea to mark every object we have access to as _owned_ to improve our visibility into more potential attack vectors. There may be a short path to our goals that hinges on ownership of a particular object. :::

Now that we informed BloodHound about our owned principals, we can run the _Shortest Paths to Domain Admins from Owned Principals_ query:

![Figura 15: Ruta más corta de BloodHound DA de los directores propios](https://offsec-platform-prod.s3.amazonaws.com/offsec-courses/PEN-200/imgs/ad_intro_enum/8dc7c1f9ec1ea297fb3443368382d3ab-bhound14.png)

Figure 15: BloodHound Shortest Path DA from Owned Principals

Let's read this by starting with the left-hand node, which is CLIENT75. As expected, _stephanie_ has a session there. The _stephanie_ user should be able to connect to CLIENT74, where _jeffadmin_ has a session. _jeffadmin_ is a part of the _Domain Admins_ group, so if we are able to take control of his account by either impersonating him or stealing the credentials on CLIENT74, we will be domain administrators.

BloodHound viene con una gran cantidad de funciones y opciones que no podemos cubrir por completo en este módulo. Si bien nos centramos principalmente en las rutas más cortas, recomendamos encarecidamente que se acostumbre a las otras consultas predefinidas de BloodHound dentro de los laboratorios de desafíos.

En este dominio en particular, pudimos enumerar la mayor parte de la información utilizando métodos manuales primero, pero en un entorno de producción a gran escala con miles de usuarios y computadoras, la información puede ser difícil de digerir. Aunque las consultas de SharpHound generan ruido en la red y es probable que sean detectadas por los analistas de seguridad, es una herramienta que vale la pena utilizar si la situación lo permite, ya que brinda una buena descripción visual del entorno en tiempo de ejecución.

Recursos
--------

Algunos de los laboratorios requieren que inicie las máquinas de destino que se indican a continuación.

Tenga en cuenta que las direcciones IP asignadas a sus máquinas de destino pueden no coincidir con las referenciadas en el texto y el video del módulo.

Enumeración de Active Directory - Bloodhound - Grupo de máquinas virtuales 1

Enumeración de Active Directory: proyecto final: grupo de máquinas virtuales 2

#### Laboratorios

1.  Si no ha recopilado datos con SharpHound en este momento, inicie el grupo de máquinas virtuales 1 y realice la recopilación de datos. Transfiera el archivo .zip generado con SharpHound a Kali Linux. Inicie BloodHound y repita los pasos de análisis descritos en esta sección para encontrar la ruta de ataque prometedora. ¿En qué servicio se basa BloodHound para mostrar los datos en gráficos?

Respuesta

Verificar

2.  Busque el grupo _Departamento de administración_ en BloodHound y use la pestaña _Información del nodo_ para ver los _derechos de control de entrada_ del grupo. ¿Quién es actualmente el propietario del grupo _Departamento de administración_ ?

Respuesta

Verificar

3.  **Ejercicio final** : inicie el grupo de máquinas virtuales 2 e inicie sesión como _Stephanie_ en CLIENT75. Desde CLIENT75, enumere los permisos de objetos para los usuarios del dominio. Una vez que se hayan identificado los permisos débiles, utilícelos para tomar el control total de la cuenta y úsela para iniciar sesión en el dominio. Una vez que haya iniciado sesión, repita el proceso de enumeración utilizando las técnicas que se muestran en este módulo para obtener la bandera.

Respuesta

Verificar

21.5. Conclusión
----------------

En este módulo, exploramos varias formas de enumerar Active Directory, cada una de las cuales aprovechaba LDAP y las clases .NET de PowerShell. Dado que Active Directory contiene una gran cantidad de información, enumerarla es un paso fundamental durante una prueba de penetración.

Los métodos de enumeración de este módulo nos brindan las habilidades básicas necesarias para realizar enumeraciones en un entorno de dominio. Si bien no podemos cubrir todas las técnicas de enumeración posibles, es fundamental profundizar en los laboratorios y explorar las clases .NET, las funciones de PowerView y las consultas BloodHound.

En el próximo módulo _Ataque de autenticación de Active Directory_ y _movimiento lateral en_ módulos de Active Directory, utilizaremos la información obtenida de este módulo y la aprovecharemos para atacar varios métodos de autenticación de Active Directory, así como para movernos lateralmente entre objetivos.

*   © 2024  [OffSec](https://www.offsec.com)
|*   [Privacidad](https://www.offsec.com/privacy-policy/)
|*   [Condiciones de servicio](https://www.offsec.com/terms-and-conditions-of-use/)

Módulo anterior

El marco de Metasploit

Siguiente módulo

Ataque a la autenticación de Active Directory
