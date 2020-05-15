# Autenticazione e autorizzazione

Procedura di autenticazione e autorizzazione token-based utilizzando AspNetCore.Identity e AspNetCore.JwtBearer.

### Installazione dei package

I seguenti package sono necessari ad autenticazione ed autorizzazione:

```
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

I seguenti package sono necessari per effettuare le migrazioni con PostgreSQL (ed eventualmente lo scaffolding):

```
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
```

Mentre il seguente è necessario per l'invio di email automatiche di conferma (facoltativo):

```
dotnet add package MailKit
```

## Autenticazione

### Modelli e DbContext

La libreria Identity fornisce già i modelli per User e Role, chiamati rispettivamente `IdentityUser<TKey>` e `IdentityRole<TKey>`, dove `TKey` è il tipo della chiave primaria (probabilmente usare `int`).
Tuttavia, è possibile estendere questi modelli se lo si vuole:

```
public class MyUser : IdentityUser<int>
{
    ...
}

public class MyRole : IdentityRole<int>
{
    ...
}
```

Per utilizzare le relazioni di Identity, bisogna che il `DbContext` eredity `IdentityDbContext<MyUser, MyRole, TKey>`:

```
public class MyDbContext : IdentityDbContext<MyUser, MyRole, TKey>
{
    ...
}
```

Che potrà essere creato a mano (code-first) o tramite scaffolding (database-first).
Nel metodo OnModelCreating inserire la seguente linea:

```
base.OnModelCreating(modelBuilder);
```

### Implementare le relazioni nei modelli User e Role (facoltativo)

Nonostante le migrazioni siano fornite di relazioni User-Role many-to-many, i modelli `IdentityUser<TKey>` e `IdentityRole<TKey>` non hanno riferimenti che puntano alle relazioni.
Un modo di implementarli potrebbe essere:

```
public class MyUserRole : IdentityUserRole<int>
{
    public MyUserRole() : base() { }

    public MyUser User { get; set; }
    public MyRole Role { get; set; } 
}
```
```
public class MyUser : IdentityUser<int>
{
    public MyUser() : base() { }

    public ICollection<MyUserRole> UserRoles { get; set; }
}
```
```
public class MyRole : IdentityRole<int>
{
    public MyRole() : base() { }
    
    public ICollection<MyUserRole> UserRoles { get; set; }
}
```

In `DbContext`, nel metodo `OnModelCreating` inserire le seguenti linee:

```
modelBuilder.Entity<MyUserRole>(entity => {
    entity.HasOne(d => d.User)
        .WithMany(p => p.UserRoles)
        .HasForeignKey(d => d.UserId);
    
    entity.HasOne(e => e.Role)
        .WithMany(u => u.UserRoles)
        .HasForeignKey(ur => ur.RoleId);
});
```

### Impostare i parametri di autenticazione

In `appsettings.json` aggiungere il seguente codice:

```
"Jwt": {
  "Issuer": "www.myawesomecompany.com",
  "SigninKey": "myverylongandsupersecuresigninkey",
  "LifeSpan": "00:30:00"
}
```

Dove `LifeSpan` è del tipo `hh:mm:ss` e rappresenta la durata del token. È possibile aggiungere altri parametri come "Audience".

### Cofigurazione dei servizi

In `Startup.cs`, nel metodo `ConfigureServices` aggiungere le identità:

```
services.AddIdentity<MyUser, MyRole>(options =>
    {
        options.Password.RequireDigit = false;
        options.Password.RequiredLength = 6;
        options.Password.RequireNonAlphanumeric = false;
        options.Password.RequireUppercase = false;
        options.Password.RequireLowercase = false;
    })
    .AddEntityFrameworkStores<MyDbContext>()
    .AddDefaultTokenProviders();
```

Dove è possibile specificare i vincoli sulla password.
Aggiungere lo schema di autenticazione:

```
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.SaveToken = true;
        options.TokenValidationParameters = new TokenValidationParameters {
            ValidateIssuer = true,
            ValidIssuer = Configuration.GetValue<string>("Jwt:Issuer"),
            ValidateAudience = false,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration.GetValue<string>("Jwt:SigninKey"))),
            ClockSkew = new TimeSpan(0),
            ValidateLifetime = true
        };
    });
```

Se non è già presente, inserire nel metodo `Configure` la seguente linea:

```
app.UseAuthorization();
```

A questo punto, è possibile proteggere le API con l'attributo `[Authorize]`.

### Configurazione dello Swagger

Aggiungere il seguente codice per abilitare l'autenticazione nello Swagger.

```
services.AddSwaggerGen(c => {
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "MyTitle", Version = "MyVersion" });
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme {
        In = ParameterLocation.Header,
        Description = "**Insert Bearer[space]JWT**",
        Name = "Authorization",
        Type = SecuritySchemeType.ApiKey
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                },
                Scheme = "oauth2",
                Name = "Bearer",
                In = ParameterLocation.Header,
            },
            new List<string>()
        }
    });;
});
```

Ciò farà comparire un pulsante di autenticazione nello Swagger, in cui inserire il token: `Bearer <JWT>`.

## Autorizzazione

In questo esempio si assumerà un'autorizzazione role-based, prevedendo 3 ruoli:

1.  SuperAdmin
2.  Admin
3.  User

In `MyDbContext`, nel metodo `OnModelCreating` (anche se sarebbe opportuno creare un metodo separato e poi richiamarlo alla fine di `OnModelCreating`), inserire i ruoli:

```
modelBuilder.Entity<Role>().HasData(
    new MyRole {
        Id = 1,
        Name = "SuperAdmin",
        NormalizedName = "SUPERADMIN"
    },
    new MyRole {
        Id = 2,
        Name = "Admin",
        NormalizedName = "ADMIN"
    },
    new MyRole {
        Id = 3,
        Name = "User",
        NormalizedName = "USER"
    }
);
```

In `OnModelCreating` impostare il valore di inizio della sequenza degli id dei ruoli, in questo caso 3 + 1.

```
modelBuilder.Entity<Role>(entity =>
    {
        entity.Property(e => e.Id)
            .HasIdentityOptions(startValue: 4);
    });
```

### Creazione delle policy di autorizzazione

In `ConfigureServices` di Startup.cs aggiungere le policy di autorizzazione:

```
services.AddAuthorization(auth => {
    auth.AddPolicy("user", new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
        .RequireAuthenticatedUser()
        .RequireRole("User")
        .Build());
    auth.AddPolicy("admin", new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
        .RequireAuthenticatedUser()
        .RequireRole("Admin")
        .Build());
    auth.AddPolicy("superadmin", new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme)
        .RequireAuthenticatedUser()
        .RequireRole("SuperAdmin")
        .Build());
```

Ora sarà possibile autorizzare le API attraverso l'attributo `[Authorize("policy")]` con `policy` che può essere `superadmin`, `admin` o `user`.
In questo esempio si presuppone che i SuperAdmin abbiano anche i ruoli di Admin e User e gli Admin abbiano anche il ruolo di User.

## Email di conferma (facoltativo)

### Configurazione appsettings

Inserire in appsettings.json le credenziali di accesso al servizio di email.

```
"Email": {
  "Name": "MyAwesomeCompany",
  "Address": "myemailaddress@company.com",
  "Password": "myverylongandsecurepassword",
  "Host": "srv-hf3.netsons.net",
  "Port": 25
}
```

### EmailHandler

Di seguito è riportato un esempio di generatore di email di conferma:

```
public class EmailHandler
{
    private readonly IConfiguration configuration;

    public EmailHandler(IConfiguration configuration)
    {
        this.configuration = configuration;
    }

    public bool SendMail(string address, string subject, string text)
    {
        try {
            // Configurazione parametri
            string senderName = configuration.GetValue<string>("Email:Name");
            string senderAddress = configuration.GetValue<string>("Email:Address");
            string senderPassword = configuration.GetValue<string>("Email:Password");
            string host = configuration.GetValue<string>("Email:Host");
            int port = configuration.GetValue<int>("Email:Port");
            
            // Impostazione messaggio
            MailboxAddress from = senderName == null ? new MailboxAddress(senderAddress) : new MailboxAddress(senderName, senderAddress);
            MimeMessage message = new MimeMessage();
            message.From.Add(from);
            message.To.Add(new MailboxAddress(address));
            message.Subject = subject;
            message.Body = new TextPart("plain") {
                Text = text
            };
            
            // Invio email
            using (var client = new SmtpClient()) {
                client.Connect(host, port, false);
                client.Authenticate(senderAddress, senderPassword);
                client.Send(message);
                client.Disconnect(true);
            }
            return true;
        } catch {
            return false;
        }
    }
}
```

## Gestione account

### AccountController

Creare il controller che conterrà le API di gestione dell'account, ovvero:

*  Register
*  Login
*  Confirm email (facoltativo)
*  Refresh Token (facoltativo)

In questo caso si chiamerà `AccountController`:

```
public class AccountController : ControllerBase
{
    private readonly UserManager<MyUser> userManager;
    private readonly IConfiguration configuration;
    
    public AccountController(UserManager<MyUser> userManager, IConfiguration configuration)
    {
        this.userManager = userManager;
        this.configuration = configuration;
    }
}
```

### Generazione token

Creare un metodo che generi il token a partire dai claim dell'utente.

```
private JwtSecurityToken GenerateTokenFromClaims(IEnumerable<Claim> claims)
{
    SymmetricSecurityKey signinKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(configuration.GetValue<string>("Jwt:SigninKey")));
    string[] hms = configuration.GetValue<string>("Jwt:LifeSpan").Split(":");
    return new JwtSecurityToken(
        issuer: configuration.GetValue<string>("Jwt:Issuer"),
        claims: claims,
        expires: DateTime.Now
            .AddHours(int.Parse(hms[0]))
            .AddMinutes(int.Parse(hms[1]))
            .AddSeconds(int.Parse(hms[2])),
        signingCredentials: new SigningCredentials(signinKey, SecurityAlgorithms.HmacSha256)
    );
}
```

Questo metodo verrà chiamato sia per generare il token durante il login che, eventualmente, per aggiornare il token.

### Registrazione

L'API per la registrazione di un nuovo utente può avere la seguente struttura, in cui si assume l'invio della email di conferma.
Il codice si basa sull'esistenza di un `RegistrationModel` che contenga almeno `UserName`, `Email` e `Password` dell'utente.

```
[HttpGet]
[Route("register")]
public async Task<IActionResult> Register(RegistrationModel model)
{
    // Costruzione dell'utente
    MyUser user = new MyUser {
        UserName = model.UserName,
        Email = model.Email
    };
    
    // Creazione dell'utente nel database
    var createResult = await userManager.CreateAsync(user, model.Password);
    if (!createResult.Succeeded) return StatusCode(500, createResult.Errors);
    
    // Aggiunta del ruolo di default
    var roleResult = await userManager.AddToRoleAsync(user, "User");
    if (!roleResult.Succeeded) return StatusCode(500, roleResult.Errors);
    
    // Generazione email di conferma (facoltativo)
    string token = await userManager.GenerateEmailConfirmationTokenAsync(user);
    string confirmationLink = Url.Action("confirmemail", "account", new { userId = user.Id, token = token }, Request.Scheme);
    EmailHandler emailHandler = new EmailHandler(configuration);
    bool emailResult = emailHandler.SendMail(model.Email, "Verify your email address",
        "To confirm your account, click on the following confirmation link: " + confirmationLink);
    if (!emailResult) return StatusCode(500, "An error occurred while sending the confirmation email");
    
    return Accepted();
}
```

### Conferma email (facoltativo)

Il seguente è un semplice esempio di conferma email tramite token.

```
[HttpGet]
[Route("confirmemail")]
public async Task<IActionResult> ConfirmEmail([Required] string userId, [Required] string token)
{
    // Ricerca utente per id
    MyUser user = await userManager.FindByIdAsync(userId);
    if (user == null) {
        return BadRequest();
    }
    
    // Conferma la mail se il token è valido
    var confirmResult = await userManager.ConfirmEmailAsync(user, token);
    if (!confirmResult.Succeeded) return BadRequest();
    
    return Ok();
}
```

### Login

Di seguito il metodo che permette il login, in cui si assume che sia necessario confermare l'email prima di potersi loggare.

```
[HttpGet]
[Route("login")]
public async Task<IActionResult> Login([Required][EmailAddress] string email, [Required] string password)
{
    // Ricerca utente per email
    MyUser user = await userManager.FindByEmailAsync(email);
    
    // Controllo che l'email sia confermata (facoltativo)
    if (user != null && !user.EmailConfirmed && await _userManager.CheckPasswordAsync(user, password)) {
        return BadRequest("Email not confirmed");
    }
    
    // Controllo che la combinazione email-password sia esatta
    var signInResult = await userManager.CheckPasswordAsync(user, password);
    if (!signInResult) return BadRequest("Invalid username or password");
    
    List<Claim> claims = new List<Claim>() {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
    };
    
    // Ruoli dell'utente
    var roles = await userManager.GetRolesAsync(user);
    foreach (string role in roles) {
        claims.Add(new Claim(ClaimTypes.Role, role));
    }
    
    // Generazione token
    JwtSecurityToken token = GenerateTokenFromClaims(claims);
    
    return Ok(new {
        token = new JwtSecurityTokenHandler().WriteToken(token),
        expires = token.ValidTo
    });
}
```

### Refresh token (facoltativo)

I token non possono essere invalidati (a meno di creare una black-list), perciò di default l'autenticazione token-base non prevede il logout.
Per minimizzare le possibilità che un token venga perso, è opportuno dotarli di una vita corta.
Per evitare che l'utente debba ripetere il login ogni volta il token scada, è possibile aggiornarlo prima della scadenza.

```
[HttpGet]
[Route("refreshtoken")]
[Authorize("user")]
public OkObjectResult RefreshToken()
{
    JwtSecurityToken newToken = GenerateTokenFromClaims(User.Claims);
    return Ok(new {
        token = new JwtSecurityTokenHandler().WriteToken(newToken),
        expires = newToken.ValidTo
    });
}
```

Ovviamente, questa API deve essere accessibile solo agli utenti loggati.
