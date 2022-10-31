# Spring Boot MapStruct İle Nesne Transferi

Bir Java developer olarak, POJO'ları haritalamanın çok katmanlı uygulamalar geliştirirken standart bir görev olduğunu kesinlikle bilirsiniz. Bu eşlemeleri manuel olarak yazmak, geliştiriciler için sıkıcı ve tatsız bir görev olmasının yanı sıra, hataya da açıktır. MapStruct, derleme sırasında güvenli ve kolay bir şekilde eşleyici sınıfı uygulamaları üreten açık kaynaklı bir Java kütüphanesidir.
Öncelikle bir SpringBoot projesi oluşturup ve aşağıda belirtilen bağımlılıkları eklemeliyiz.

```xml
<dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct-processor</artifactId>
      <version>${mapstruct.version}</version>
    </dependency>
    
    
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>${maven-compiler-plugin.version}</version>
        <configuration>
          <release>${java.version}</release>
          <annotationProcessorPaths>
            <path>
              <groupId>org.mapstruct</groupId>
              <artifactId>mapstruct-processor</artifactId>
              <version>${mapstruct.version}</version>
            </path>
            <path>
              <groupId>org.projectlombok</groupId>
              <artifactId>lombok</artifactId>
              <version>${lombok.version}</version>
            </path>
          </annotationProcessorPaths>
        </configuration>
      </plugin>

```
## Uygulama Örneği

Doktor, Uzmanlık ve Doktor uzmanlıkları olan 3 tablo üzerinden bir örnek geliştirelim.

## _Entity Sınıflarımız_

Profession .java
```java
public class Profession extends BaseEntity {
   @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String description;
    @ManyToOne
    @JoinColumn(name="parent_id", nullable = true)
    private Profession parent;

}
``` 

Doctor.java
```java
public class Doctor extends BaseEntity {
    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String surname;
    private String  email;
    private String  phone;

    @ManyToMany(cascade = CascadeType.ALL)
    @JoinTable(
            name="doctor_professions",
            joinColumns = @JoinColumn( name="doctor_id"),
            inverseJoinColumns = @JoinColumn( name="profession_id")
    )
    private List<Profession> professionList;
}
```
DTO sınıflarımız
DoctorRequestDto.java
```java
public class DoctorRequestDto implements Serializable {
    private Long id;
    private String doctorName;
    private String doctorSurName;
    private String phone;
    private List<Long> professionIdList;

}
```

DoctorResponseDto.java
```java
public class DoctorResponseDto implements Serializable {
    private Long id;
    private String externalId;
    private String doctorName;
    private String doctorSurName;
    private String phone;
    private List<ProfessionResponseDto> professionResponseDtos;
}
```

ProfessionRequestDto.java
```java
public class ProfessionRequestDto implements Serializable {
    private Long id;
    private Long parentId;
    private String name;
    private String description;
}
```

ProfessionResponseDto.java
```java
public class ProfessionResponseDto implements Serializable {
   private Long id;
   private Long parentId;
    private String name;
    private String description;
    private String  parentName;
}
```
Mapper Sınıflarımız
ProfessionMapper.java
```java

@Mapper(componentModel = "spring")
public interface ProfessionMapper {
    @Mapping(source = "parent.id", target = "parentId")
    @Mapping(source = "parent.name", target = "parentName")
    ProfessionResponseDto toDto(Profession profession);

    List<ProfessionResponseDto> toDto(List<Profession> profession);

    @Mapping(source = "parentId", target = "parent.id")
    @Mapping(target = "deleted",constant = "false")
    @Mapping(target = "id",ignore = true)
    Profession toEntity(ProfessionRequestDto professionRequestDTO);
    
    @Mapping(source = "id", target = "id")
    Profession toEntity(Long id);

    @Mapping(source = "parentId", target = "parent.id")
    @Mapping(target = "deleted",ignore = true)
    void updateEntity(@MappingTarget Profession profession, ProfessionRequestDto professionRequestDTO);

    @BeforeMapping
    default void doBeforeMapping(@MappingTarget Profession entity, ProfessionRequestDto dto) {
        entity.setParent(null);
    }
    @AfterMapping
    default void doAfterMapping(@MappingTarget Profession entity, ProfessionRequestDto dto) {
         if (dto.getParentId() == null) {
            entity.setParent(null);
        }
    }

}

```
DoctorMapper.java
```java

@Mapper(componentModel = "spring", imports = UUID.class, uses = {ProfessionMapper.class})
public interface DoctorMapper {
    @Mapping(source = "name", target = "doctorName")
    @Mapping(source = "surname", target = "doctorSurName")
    @Mapping(source = "professionList", target = "professionResponseDtos")
    @Mapping(target = "externalId", expression = "java(UUID.randomUUID().toString())")
    DoctorResponseDto toDto(Doctor doctor);

    List<DoctorResponseDto> toDto(List<Doctor> doctor);

    @Mapping(source = "doctorName", target = "name")
    @Mapping(source = "doctorSurName", target = "surname")
    @Mapping(source = "professionIdList", target = "professionList")
    @Mapping(target = "deleted",constant = "false")
    @Mapping(target = "id",ignore = true)
    Doctor toEntity(DoctorRequestDto doctorRequestDTO);

    @Mapping(source = "doctorName", target = "name")
    @Mapping(source = "doctorSurName", target = "surname")
    @Mapping(source = "professionIdList", target = "professionList")
    @Mapping(target = "deleted",constant = "false")
    void updateEntity(@MappingTarget Doctor doctor,DoctorRequestDto doctorRequestDTO);

    @BeforeMapping
    default void doBeforeMapping(@MappingTarget Doctor entity, DoctorRequestDto dto) {
        entity.setProfessionList(null);
    }

    @AfterMapping
    default void doAfterMapping(@MappingTarget Doctor entity, DoctorRequestDto dto) {
        if (dto.getProfessionIdList() == null) {
            entity.setProfessionList(null);
        }
    }
}
```

## _`Mapstructta dikkat edilmesi gerekenler`_
-	Sınıfın başına `@Mapper` notasyonu yazmalıyız. `componentModel = “spring”` yazarak bean haline gelmektedir Başka yerler de autowired etmemizi sağlamaktadır.
-	Aynı isimde olan propertyleri kendisi eşler.

-	Farklı isimleri kendimiz eşleştirmeliyiz. `@Mapping(source = "doctorName", target = "name")`

-	Child sınıfa bir değer atamak istiyorsak ; `@Mapping(source = "parent.name", target = "parentName")`

-	Child sınıfın tamamını kullanmak istiyorsak, o sınıfın mapperini uses içinde tanımlayarak kullanabiliriz; `@Mapper(componentModel = "spring", uses = {ProfessionMapper.class})`
-
DoctorMapperin professionMapperi kullanarak professionu eşleştirdiğini aşağıdan görebiliriz;

```java
@Override
public DoctorResponseDto toDto(Doctor doctor) {
    if ( doctor == null ) {
        return null;
    }

    DoctorResponseDto doctorResponseDto = new DoctorResponseDto();

    doctorResponseDto.setDoctorName( doctor.getName() );
    doctorResponseDto.setDoctorSurName( doctor.getSurname() );
    doctorResponseDto.setProfessionResponseDtos( professionMapper.toDto( doctor.getProfessionList() ) );
    doctorResponseDto.setId( doctor.getId() );
    doctorResponseDto.setPhone( doctor.getPhone() ); 

    return doctorResponseDto;
}
```

-	 Bir değer atarken `qualifiedByName` ve `@Named` notasyonu ile özel method kullanabilirsiniz;

```java
@Mapping(
        source = "professionList",
        target = "numProfessions",
        qualifiedByName = "countProfessions") 
DoctorResponseDto toDto(Doctor doctor);

@Named("countProfessions")
default int getProfessionsSize(List<Profession> professions) {
    if(professions == null) {
        return 0;
    }
    return professions.size();
}
```
- Mapstruct'ta `@BeforeMapping`'i kullanarak, haritalama gerçekleşmeden hemen önce oluşturduğumuz kodu çalıştırabileceğiz.

- Mapstruct'ta `@AfterMapping` kullanımı, mapping yapıldıktan sonra yürütülecek, böylece mapping bittikten sonra değerleri atayabiliriz.

```java
@BeforeMapping
default void doBeforeMapping(@MappingTarget Doctor entity, DoctorRequestDto dto) {
    entity.setProfessionList(null);
}

@AfterMapping
default void doAfterMapping(@MappingTarget Doctor entity, DoctorRequestDto dto) {
    if (dto.getProfessionIdList() == null) {
        entity.setProfessionList(null);
    }
}

```
-	`Constant` notasyonu kullanarak bir değişkene sabit olarak bir değer setleyebiliriz.
```sh
 @Mapping(target = "deleted",constant = "false")
```
-	Java expression kullanmak
```sh
@Mapping(target = "externalId", expression = " java( UUID.randomUUID( ).toString())")
```
Bu notasyon sayesinde istenilen java expressionları kullanılabilinir.

Örneklerde görüldüğü gibi, Mapstruct basit ve karmaşık haritalayıcıları kolay ve hızlı bir şekilde oluşturmamıza izin veren çok sayıda işlevsellik ve konfigürasyon sunar. Mapstruct diğer mapper ile kıyaslamasına şuradan ulaşabilirsiniz (https://www.baeldung.com/java-performance-mapping-frameworks).

![](https://github.com/erogluhasan/spring-mapstruct-demo/tree/main/src/main/resources/1.png?raw=true)

Uygulamanın tüm kodlarına şuradan ulaşabilirsiniz: [Github](https://github.com/erogluhasan/springboot-hazelcast-example)

 
