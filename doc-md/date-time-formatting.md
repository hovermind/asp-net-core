## For API DTO
##### Create converter to set format   
`IsoDateConverter.cs`
```cs
public class IsoDateConverter : IsoDateTimeConverter
{
    public IsoDateConverter() => this.DateTimeFormat = "yyyy/MM/dd";
}
```

To support culture (see: [this](https://stackoverflow.com/a/47699340/4802664))
```cs
public class IsoDateConverter : IsoDateTimeConverter
{
    public IsoDateConverter() => this.DateTimeFormat = Culture.DateTimeFormat.ShortDatePattern;
}
```

##### Apply converter to property   
`CreateEmployeeDto.cs`
```cs
public class CreateEmployeeDto
{
    [Required]
    [MinLength(3)]
    public string Name { get; set; }
    
    [Required]
    [JsonConverter(typeof(IsoDateConverter))]
    public DateTime? DateOfBirth { get; set; }

    public string Address { get; set; }
}
```

## For Web App DTO
##### Use DisplayFormat to DateTime
`EmployeeDto.cs`
```cs
public class EmployeeDto
{
    [Required]
    [MinLength(3)]
    public string Name { get; set; }

    [DisplayFormat(ApplyFormatInEditMode = true, DataFormatString = "{0:yyyy-MM-dd}", ConvertEmptyStringToNull = true)]
    [Required]
    public DateTime? DateOfBirth { get; set; }

    public string Address { get; set; }
}
```

##### Use DisplayFormat to String
```cs
public class EmployeeDto
{
    [Required]
    [MinLength(3)]
    public string Name { get; set; }

    [DataType(DataType.Date)]
    [DisplayFormat(ApplyFormatInEditMode = true, DataFormatString = "{0:yyyy-MM-dd}", ConvertEmptyStringToNull = true)]
    [Required]
    public string DateOfBirth { get; set; }

    public string Address { get; set; }
}
```

## Global Custom Datetime Formatting
`Startup.cs`
```cs
var config = GlobalConfiguration.Configuration;
var settings = new JsonSerializerSettings();
settings.Converters.Add(new IsoDateTimeConverter()
	{
		DateTimeFormat = "dd.MM.yyyy"
	});
config.Formatters.Remove(config.Formatters.JsonFormatter);
config.Formatters.Add(new JsonNetFormatter(settings)); 
```

Another way
```
var jsonFormatter = config.Formatters.JsonFormatter;
jsonFormatter.SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc;
```

See:
* https://gist.github.com/AlexZeitler/2821442
* http://www.hackered.co.uk/articles/asp-net-web-api-controlling-date-formats-with-json-net
* https://forums.asp.net/t/2084573.aspx?UTC+Date+is+being+deserialized+to+server+local+time+when+PUT+and+PATCH+
