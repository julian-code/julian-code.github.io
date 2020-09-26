---
layout: post
title:  "UnitTesting med xUnit, Moq og AutoFixture"
date:   2020-09-26
categories: jekyll update
---
På studiet har vi et fag om testing, og her har fokus været på at unit teste sin kode med xUnit. Her har vi kigget på mocking, test drevet udvikling osv. Her er hvad jeg har undersøgt, samt fået ud af undervisningen.

TL;DR
=====
Vi har kigget på test drevet udvikling, samt hvordan det spiller sammen med Moq og AutoFixture og hvordan det kan hjælpe dig med at skrive mere robuste tests. 

Data-Drevet testing med xUnit
=====

Her har vi haft fokus på xUnits data drevet test attribut Theory. Theory giver dig mulighed for,
at definere paramateriseret tests for at gøre dine test data drevet.

```csharp
[Theory]
public void FooTest(int a, int b)
{
  //.. 
}
```

For at kunne sende data ind som parametre bruges der tre forskellige typer af attributter: InlineData, MemberData og ClassData.

En InlineData er relativt simpelt og defineres således:

```csharp
[Theory]
[InlineData(1,2)]
public void FooTest(int a, int b)
{
  //.. 
}
```
Ved dette sættes argument a som 1, og argument b som 2. 

En anden måde hvorpå man kan sende data ind, er via. MemberData.

```csharp
public class MyFooTests 
{
  [Theory]
  [MemberData(nameof(MyMemberData))]
  public void FooTest(int a, int b) 
  {

  }

  public static IEnumerable<object[]> MyMemberData => 
  {
    new List<object[]> 
    {
      new object[] {1,2}
    }
  }
}
```

En anden måde er ClassData hvor du definere en helt klasse til at indeholde din test data.

```csharp
public class MyFooTests 
{
  [Theory]
  [ClassData(typeof(FooTestTestData))]
  public void FooTest(int a, int b) 
  {

  }
}

public class FooTestTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { 1, 2 };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

Alle disse tests får fuldstændig de samme parametre ind som a og b, bare på tre forskellige måder. 

En ting jeg faldt over var brugen af keywordet yield. yield fortæller compileren at når der foreaches over FooTestTestData, at den skal holde på næste reference den skal returnere.
Det betyder at den vil itererer over de indeholdende elementer på den måde de er defineret.
En anden smart måde er faktisk at hvis du har yields kan du foreache over værdierne. Se følgende eksempel fra Microsoft Docs:

```csharp
public static class GalaxyClass
{
    public static void ShowGalaxies()
    {
        var theGalaxies = new Galaxies();
        foreach (Galaxy theGalaxy in theGalaxies.NextGalaxy)
        {
            Debug.WriteLine(theGalaxy.Name + " " + theGalaxy.MegaLightYears.ToString());
        }
    }

    public class Galaxies
    {

        public System.Collections.Generic.IEnumerable<Galaxy> NextGalaxy
        {
            get
            {
                yield return new Galaxy { Name = "Tadpole", MegaLightYears = 400 };
                yield return new Galaxy { Name = "Pinwheel", MegaLightYears = 25 };
                yield return new Galaxy { Name = "Milky Way", MegaLightYears = 0 };
                yield return new Galaxy { Name = "Andromeda", MegaLightYears = 3 };
            }
        }
    }

    public class Galaxy
    {
        public String Name { get; set; }
        public int MegaLightYears { get; set; }
    }
}
```

Nå tilbage til hvor vi slap - ClassMember, InlineData og MemberData, hvad har de tilfælles?

Vi se at MemberData og ClassData forventer en eller anden form for IEnumerable<object[]>. Men hvordan konverteres de to argumenter til et objekt array i vores InlineData?

Lad os se på InlineDataAttribute fra xUnits kildekode. 

```csharp
/// <summary>
/// Provides a data source for a data theory, with the data coming from inline values.
/// </summary>
[DataDiscoverer(typeof(InlineDataDiscoverer))]
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public sealed class InlineDataAttribute : DataAttribute
{
  readonly object?[] data;

  /// <summary>
  /// Initializes a new instance of the <see cref="InlineDataAttribute"/> class.
  /// </summary>
  /// <param name="data">The data values to pass to the theory.</param>
  public InlineDataAttribute(params object?[] data)
  {
    this.data = data;
  }

  /// <inheritdoc/>
  public override IEnumerable<object?[]> GetData(MethodInfo testMethod) => new[] { data };
}
```
Her ser man brugen af params keywordet i konstruktøren. Det gør at man kan angive ens parametre til at være komma separeret. 

Når man kigger på deres kilde kode, kan man se at de tre arver fra DataAttribute. DataAttribut er en abstrakt klasse, som tvinger dig til at implementere GetData som tager imod en MethodInfo.
MethodInfo er fundet i System.Reflection og er en måde hvorpå man kan få informationer ud af en metode fra en type, f.eks. parametrene metoden bliver kaldt med.

Dette betyder altså at du selv kan definere hvordan data skal komme fra, og hvad der skal ske inden du får data ved blot at implementere din egen DataAttribute. 

Her ses et eksempel på en implementation som kan læse fra en CSV fil.

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = false)]
public class ReadCsvData : DataAttribute
{
    private readonly string filePath;
    private readonly bool hasHeaders;

    public ReadCsvData(string filePath, bool hasHeaders = false)
    {
        this.filePath = filePath;
        this.hasHeaders = hasHeaders;
    }
    public override IEnumerable<object[]> GetData(MethodInfo testMethod)
    {
        var methodParameters = testMethod.GetParameters();

        var paramenterTypes = methodParameters.Select(x => x.ParameterType).ToArray();

        using (var streamReader = new StreamReader(filePath))
        {
            if (hasHeaders)
            {
                streamReader.ReadLine();
            }
            string csvLine = string.Empty;
            while ((csvLine = streamReader.ReadLine()) != null)
            {
                var csvRow = csvLine.Split(',');
                yield return ConvertFromCsv((object[])csvRow, paramenterTypes);
            }
        }
    }

    private object[] ConvertFromCsv(IReadOnlyList<object> csvRow, IReadOnlyList<object> paramenterTypes)
    {
        var convertedObject = new object[paramenterTypes.Count];

        for (int i = 0; i < paramenterTypes.Count; i++)
        {
            convertedObject[i] = (paramenterTypes[i] == typeof(string)) ? Convert.ToString(csvRow[i]) : csvRow[i];
        }

        return convertedObject;
        
    }
}
```

Dette betyder faktisk, at man kan gøre sin CI/CD pipeline utrolig smart ved at have et flow hvor vitale dele af ens software gemmer fejlede statements ned, som så bliver ført igennem parametre til testen. På den måde gør du din kode virkelig test drevet - du kan ikke release noget før de fejl, som er sket i produktion, er fikset.

Mock med Moq
=====
For at man kan skrive testbar kode er det meget smart ikke at have for mange afhængigheder.

Et eksempel på en afhængighed kunne være følgende:

```csharp
public class Foo 
{
  Bar bar;

  public Foo() 
  {
    this.bar = new Bar()
  }
}

public class Bar 
{

}
```

Her har Foo en direkte afhængighed til Bar. Hvis Bar skifter - jamen så skifter opførelsen af Foo også. Hvordan kan vi komme udenom dette? Ved brug af f.eks. interfaces.

```csharp
public class Foo 
{
  IBar bar;

  public Foo(IBar bar) 
  {
    this.bar = bar
  }
}

public class Bar : IBar
{

}

public interface IBar {}
```

Nu er det eksternt kode der skal sørge for, hvordan opførelsen af Foo skal være - det betyder at vi har vendt afhængigheden om (Dependency Inversion).

Men, hvordan tester man dette? Man kan jo ikke instantiere et interface, og hvis jeg gør, så er det jo på en konkret implementering som kan gøre alt muligt jeg ikke vil have den til?

Vi prøver lige med et andet eksempel for at illustrere pointe bedre.

```csharp
public class Service 
{
  IRepository repository;

  public Foo(IRepository repository) 
  {
    this.repository = repository
  }

  public Person UpdatePerson(Person person) {
    var person = repository.GetById(person.Id);
    //Opdaterings logik.
    return updatedPerson;
  }
}

public class Repository : IRepository
{
  public Person GetById(int id) 
  {
    // Data logic
    return person;
  }
}

public interface IRepository 
{
  Person GetById(int id)
}

public class Person 
{
  public int Id {get; set;}
  public string FirstName {get; set;}
  public string LastName {get; set;}
}

```

Meget simpelt eksempel - en service henter fra et repository, og opdaterer personen.

Hvis vi skulle skrive en test, ville det se således ud

```csharp
[Fact]
public void Service_UpdatePerson() 
{
  //Arrange
  var repository = new Repository();
  var sut = new Service(repository);

  //Act
  var person = sut.UpdatePerson(new Person { Id = 1, FirstName = "UpdatedFirstName", LastName = "UpdatedLastName"});

  //Assert tjek om personen er lig med den opdateret person.
}
```

Der er tre ting der er forkerte i denne test.
* Der er afhængighed i vores test til Repository.
* Der er potentiale for, at vi kalder noget forkert når vi bruger en konkret implementering - det kunne være vi kaldte noget der kunne skade produktion.
* Klasserne der testes på kan ændre sig, f.eks. konstruktøren eller property navne (mere om det senere)

Hvordan løser vi det her? Med Mocking - mocking er hvor man overstyrer sine objekter til at fortælle dem, hvordan de skal opføre sig.

En normal Mock kunne laves således:

```csharp
public class TestRepository : IRepository 
{
    public Person GetById(int id) 
    {
      return new Person {
        Id = id;
        FirstName = "NotUpdatedFirstName",
        LastName = "NotUpdatedLastName"
        //... sæt resterende via test konstanter.
      }
      return person;
    }
}
```

Nu har vi overstyret repositoriet til at gøre, lige præcis som vi vil have det skal gøre - men det bliver en meget omstændelig proces at gøre for alle objekter der skal mockes.

Derfor findes der Moq.

Med dette kunne vi overstyrer processen helt ved følgende:

```csharp
[Fact]
public void Service_UpdatePerson() 
{
  //Arrange
  var repository = new Mock<IRepository>();
  repository.Setup(x => x.GetById(It.IsAny<int>)).Returns(new Person {Id = 1, FirstName = "NotUpdatedFirstName", LastName = "NotUpdatedLastName"})
  var sut = new Service(repository.Object);

  //Act
  var person = sut.UpdatePerson(new Person { Id = 1, FirstName = "UpdatedFirstName", LastName = "UpdatedLastName"});

  //Assert tjek om GetById er blevet kaldt hvordan?
}
```
Vi starter med at instantiere en Mock af Repository. På den måde har vi nu abstraheret konkrete implementeringer væk, fordi Mock har essentielt lavet TestRepository for os. Dette gøres ved hjælp
af en dynamisk proxy som essentiel compiler og opretter en klasse på runtime, som har den opførelse du sætter til at være gældende i f.eks. Setup. Hvis du vil læse mere om dynamiske proxyer så gør
Moq brug af http://www.castleproject.org/projects/dynamicproxy/

Setup fortæller at lige præcis denne her metode på Mocken skal returnere dette uanset hvad den bliver kaldt med (It.IsAny).

Nå men nu har vi fjernet to fejl
* Der er afhængighed i vores test til Repository.
* Der er potentiale for, at vi kalder noget forkert når vi bruger en konkret implementering - det kunne være vi kaldte noget der kunne skade produktion.

Men hvad med den sidste?

Data Drevet Testing med AutoFixture og AutoMoq
=====

```csharp
[Fact]
public void Service_UpdatePerson() 
{
  //Arrange
  var repository = new Mock<IRepository>();
  repository.Setup(x => x.GetById(It.IsAny<int>)).Returns(new Person {Id = 1, FirstName = "NotUpdatedFirstName", LastName = "NotUpdatedLastName"})
  var sut = new Service(repository.Object);

  //Act
  var person = sut.UpdatePerson(new Person { Id = 1, FirstName = "UpdatedFirstName", LastName = "UpdatedLastName"});

  //Assert tjek om personen er lig med den opdateret person.
}
```
I den overstående test er stadig følgende fejl.

* Klasserne der testes på kan ændre sig, f.eks. konstruktøren eller property navne (mere om det senere)

Og hvad gør vi hvis Service får nye dependencies? 

Her kan AutoFixture og AutoMoq hjælpe os

AutoFixture hjælper os helt vildt med vores Arrange del. Lad os kigge på hvordan.

```csharp
[Theory]
[AutoMoqData]
public void Service_UpdatePerson(Person personToUpdate, [Frozen] Mock<IRepository> repository, Service sut) 
{
  //Act
  var person = sut.UpdatePerson(personToUpdate);

  //Assert tjek om personen er lig med den opdateret person.
  repository.Verify(x => x.GetById(personToUpdate.Id), Timens.Once);
}

[AttributeUsage(AttributeTargets.Method)]
public sealed class AutoMoqDataAttribute : AutoDataAttribute 
{
  public AutoMoqDataAttribute() : base(() => Fixture().Customize(new AutoMoqCustomization()))
}
```

AutoFixture og AutoMoq søger for at personToUpdate's properties bliver fyldt ud - på den måde slipper vi for at holde styr på deres f.eks. constructors og felters navne når vi instantiere.
Ved brug af AutoMoq mocker den alle Service's dependencies - hvis vi skal fange en dependency, for f.eks. at Verify noget bliver kaldt skal vi definere det i parametrene med Frozen og så mocken.

Der er selvfølgelig en stor brug af Reflection her - så det kommer med en pris på performance af dine tests - men tilgengæld får du meget tydelige og utrolig fleksible test. 

Kilder:

[xUnit](https://github.com/xunit/xunit)

[AutoFixture](htps://github.com/AutoFixture/AutoFixture)

[moq](https://github.com/moq/moq4)