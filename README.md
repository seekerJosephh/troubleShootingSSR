# troubleShootingSSR

// Controller.cs 

using it_troubleshooting_service.AppContext;
using it_troubleshooting_service.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System.Net;

namespace it_troubleshooting_service.Controllers
{

    [ApiController]
    [Route("api/[controller]")]
    public class TroubleController : ControllerBase
    {
        private readonly MyDbContext _context;
        private readonly IWebHostEnvironment _environment;

        public TroubleController(MyDbContext context, IWebHostEnvironment environment)
        {
            _context = context;
            _environment = environment;
        }

        [HttpGet("apps")]
        public async Task<IActionResult> GetApps()
        {
            var apps = await _context.TblAppMst
                .Select(a => new { Value = a.AppId, ViewValue = a.AppName })
                .ToListAsync();
            return Ok(apps);
        }


        [HttpGet("device-name")]
        public IActionResult GetDeviceName()
        {
            string ip = HttpContext.Connection.RemoteIpAddress?.ToString();

            if (string.IsNullOrEmpty(ip))
                return BadRequest(new { error = "Unable to detect client IP" });

            if (ip.StartsWith("::ffff:"))
                ip = ip.Replace("::ffff:", "");

            string hostName = null;

            try
            {
                var hostEntry = System.Net.Dns.GetHostEntry(ip);
                hostName = hostEntry.HostName;
            }
            catch
            {
                hostName = null;
            }

            return Ok(new { ip, deviceName = hostName });
        }



        [HttpPost]
        [Consumes("multipart/form-data")]   
        public async Task<IActionResult> PostTrouble([FromForm] CreateTroubleRequest request)
        {
            string? errorImagePath = null;

            if (request.ErrorImage != null)
            {
                var uploadsFolder = Path.Combine(_environment.WebRootPath, "uploads");
                if (!Directory.Exists(uploadsFolder))
                {
                    Directory.CreateDirectory(uploadsFolder);
                }
                var uniqueFileName = Guid.NewGuid().ToString() + "_" + request.ErrorImage.FileName;
                var filePath = Path.Combine(uploadsFolder, uniqueFileName);
                using (var fileStream = new FileStream(filePath, FileMode.Create))
                {
                    await request.ErrorImage.CopyToAsync(fileStream);
                }
                errorImagePath = "/uploads/" + uniqueFileName;
            }

            var trouble = new TroubleRec
            {
                StartDate = DateTime.UtcNow,
                AppId = request.AppId,
                PlantId = request.PlantId,
                Location = request.Location,
                HostName = request.HostName,
                IpAddr = request.IpAddress,           
                IssueDesc = request.IssueDesc,
                ErrorImage = errorImagePath,
                IsClose = false,
                CreatedBy = "SystemUser",
                CreatedDate = DateTime.UtcNow,
            };

            _context.TblTroubleRec.Add(trouble);
            await _context.SaveChangesAsync();

            return Ok(new { Message = "Trouble record created successfully", Id = trouble.IssId });
        }

        private string GetClientHostname()
        {
            try
            {
                var ipAddress = HttpContext.Connection.RemoteIpAddress?.ToString();
                if (string.IsNullOrEmpty(ipAddress))
                {
                    return "Unknow";
                }
                var hostEntry = Dns.GetHostEntry(ipAddress);
                return hostEntry.HostName;
            }catch (Exception ex)
            {
                return "Hostname lookup failed";
            }

        }
    }
}

//  AppContext 


using it_troubleshooting_service.Models;
using Microsoft.EntityFrameworkCore;

namespace it_troubleshooting_service.AppContext
{
    public class MyDbContext : DbContext 
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options) { }

        public DbSet<TroubleRec> TblTroubleRec { get; set; }
        public DbSet<AppMst> TblAppMst { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<TroubleRec>().ToTable("tbltrouble_rec");

            modelBuilder.Entity<AppMst>().ToTable("tblapp_mst");

            
            modelBuilder.Entity<AppMst>()
                .Property(a => a.AppName)
                .HasColumnName("App Name");   

            
            modelBuilder.Entity<AppMst>()
                .Property(a => a.AppDesc)
                .HasColumnName("App Desc");

            modelBuilder.Entity<AppMst>()
                .Property(a => a.DevelopLang)
                .HasColumnName("develop lang");

            
            modelBuilder.Entity<AppMst>()
                .Property(a => a.AppId)
                .HasColumnName("appid");

           
            modelBuilder.Entity<AppMst>()
                .HasKey(a => a.AppId);
        }
    }
}

// Model 

using System.ComponentModel.DataAnnotations;

namespace it_troubleshooting_service.Models
{
    public class AppMst
    {
        [Key]
        public int AppId { get; set; }
        public string? AppName { get; set; }
        public string? AppDesc { get; set; }
        public int? ApCatId { get; set; }
        public int? ApSrcId { get; set; }
        public string? Process { get; set; }
        public string? Url { get; set; }
        public int? CritlId { get; set; }
        public int? SvrId { get; set; }
        public string? DevelopLang { get; set; }
        public string? Authentication { get; set; }
        public string? CreatedBy { get; set; }
        public DateTime? CreatedDate { get; set; }
        public string? UpdatedBy { get; set; }
        public DateTime? UpdatedDate { get; set; }
    }
}

namespace it_troubleshooting_service.Models
{
    public class CreateTroubleRequest
    {
        public int AppId { get; set; }
        public int PlantId { get; set; }
        public string? Location { get; set; }
        public string? HostName { get; set; }
        public string? IpAddress { get; set; }
        public string? IssueDesc { get; set; }
        public IFormFile? ErrorImage { get; set; }
    }
}

using System.ComponentModel.DataAnnotations;

namespace it_troubleshooting_service.Models
{
    public class TroubleRec
    {
        [Key]
        public int IssId { get; set; }
        public DateTime? StartDate { get; set; }
        public int AppId { get; set; }
        public int PlantId { get; set; }
        public string? Location { get; set; }
        public string? HostName { get; set; }
        public string? IpAddr { get; set; }
        public string? IssueDesc { get; set; }
        public int? InfEmpId { get; set; }
        public int? IssEmpId { get; set; }
        public DateTime? ClosedDate { get; set; }
        public int? CloseEmpId { get; set; }
        public string? SolutionDesc { get; set; }
        public string? ErrorImage { get; set; }
        public bool? IsClose { get; set; }
        public string? RelatedTeam { get; set; }
        public string? Url { get; set; }
        public string? CreatedBy { get; set; }
        public DateTime? CreatedDate { get; set; }
        public string? UpdatedBy { get; set; }
        public DateTime? UpdatedDate { get; set; }
    }
}

using it_troubleshooting_service.AppContext;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);


builder.Services.AddControllers();

builder.Services.AddDbContext<MyDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Server")   
               ?? throw new InvalidOperationException("Connection string 'server' not found.")
    ));

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
        
    });
});

var app = builder.Build();


if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();    
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseCors("AllowAll");

app.UseAuthorization();

app.MapControllers();

app.Run();

// and here is table look like 

SELECT TOP (1000) [issid]
      ,[Start Date]
      ,[appid]
      ,[plantid]
      ,[Location]
      ,[Host Name]
      ,[IP Addr.]
      ,[Issue Desc]
      ,[inf_empid]
      ,[iss_empid]
      ,[Closed Date]
      ,[close_empid]
      ,[Solution Desc]
      ,[Error Image]
      ,[isclose]
      ,[relatedteam]
      ,[url]
      ,[Created By]
      ,[Created Date]
      ,[Updated By]
      ,[Updated Date]
  FROM [ittrouble_db].[dbo].[tbltrouble_rec]
  
  
  // Frontend 
  
  method OnSubmit()
  
    onSubmit() {
    const formData = new FormData();
    formData.append('AppId', this.selectedApp?.toString() || '');
    formData.append('PlantId', this.selectedPlant?.toString() || '');
    formData.append('Location', this.location);
    formData.append('HostName', this.hostName);
    formData.append('IpAddress', this.ipAddress);
    formData.append('IssueDesc', this.problems);
    if (this.errorImageFile) {
      formData.append('ErrorImage', this.errorImageFile);
    }

    this.dataService.postTrouble(formData).subscribe(response => {
      console.log('Success:', response);
      
    }, (error: any) => {
      console.error('Error:', error);
    });
  }

is that doesn't have method post data into DB, to handle submit() in fontend ?
