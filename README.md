# FleetCheck – Build Systems Lab

This project is intentionally incomplete. Follow the worksheet in the order given.

Expected final application output:

```
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

Do not copy the solution POM. The objective is to observe how each build change alters the result.
## Evidence 1
[ERROR] /C:/Users/Diogo/Downloads/FleetCheck_Starter/FleetCheck_Starter/src/main/java/pt/upt/fleetcheck/App.java:[4,38] package com.fasterxml.jackson.databind does not exist
O erro vem do import na linha 4 do App.java: `import com.fasterxml.jackson.databind.ObjectMapper;`

## Step 2 – Adicionar o jackson-databind

Adicionei a dependência `jackson-databind` 2.22.2 ao `pom.xml` e corri `mvn clean package`.

**Resultado:** a compilação passou (`BUILD SUCCESS`), ao contrário do Passo 1.

**Desvio em relação ao esperado:** a ficha indica que a fase de testes devia falhar.
No meu caso isso não aconteceu, porque o projeto starter não inclui nenhuma classe
de teste (a pasta `src/test/java/pt/upt/fleetcheck` está vazia). O Surefire não
encontrou testes para executar, por isso não detetou nenhum defeito de comportamento.

## Step 3 – Dependency tree

Executei `mvn dependency:tree`. O `jackson-databind` 2.22.2 é a única dependência
direta de aplicação. O `jackson-core` e o `jackson-annotations` aparecem como
dependências transitivas, trazidas automaticamente pelo `jackson-databind`.
O JUnit tem escopo `test`, por isso não faz parte do artefacto final.

## Step 4 – JAR executável

**JAR normal** (`mvn clean package` + `java -jar target/fleetcheck-1.0.0.jar`):

    no main manifest attribute, in target/fleetcheck-1.0.0.jar

**Com o maven-shade-plugin** (`mvn clean package` + `java -jar target/fleetcheck-1.0.0-all.jar`):

    FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km

**Evidence 4 – O que o Shade mudou em relação ao JAR por omissão:**
O JAR normal contém apenas as classes do projeto, não declara a `Main-Class` no manifesto
e não inclui o Jackson, por isso não é executável sozinho. O Shade gera um JAR "gordo"
(`fleetcheck-1.0.0-all.jar`) que copia para dentro as classes das dependências (Jackson)
e escreve a `Main-Class` (`pt.upt.fleetcheck.App`) no manifesto, tornando-o autónomo.

## Step 5 – Maven Wrapper

Gerei o wrapper com `mvn wrapper:wrapper`, que criou `mvnw`, `mvnw.cmd` e `.mvn/wrapper/`,
e fiz commit destes ficheiros. Marquei também o `mvnw` como executável no Git
(`git update-index --chmod=+x mvnw`) para funcionar em Linux. Adicionei ao `pom.xml` a
propriedade `project.build.outputTimestamp`, que dá aos plugins compatíveis um timestamp
fixo nos artefactos. Executei `.\mvnw.cmd clean verify` com `BUILD SUCCESS`.

## Step 6 – GitHub Actions (Maven)

Criei o workflow `.github/workflows/build.yml`, que faz checkout, instala o JDK 21,
corre `./mvnw -B clean verify` e carrega o artefacto `fleetcheck-build`.

URL da execução com sucesso: https://github.com/diogogarcia09/fleetcheck/actions/runs/37235936882

## Passo 7 – SBOM (CycloneDX)

Adicionei o `cyclonedx-maven-plugin` à fase `verify`. Depois de `.\mvnw.cmd clean verify`
foi gerado `target/bom.json`, onde encontrei `jackson-databind`, `jackson-core` e `jackson-annotations`.

**Evidence 7 – Porque é que o SBOM contém componentes que não escrevi?**
O SBOM lista todas as dependências que fazem parte do software, e não só as que declarei.
Eu declarei apenas o `jackson-databind`, mas ele depende do `jackson-core` e do
`jackson-annotations`, que o Maven resolve como dependências transitivas. Como o SBOM
descreve a cadeia completa de componentes (útil para auditar vulnerabilidades e licenças),
estas dependências também aparecem lá.

## Evidence 8.1

Comando: `gradle clean build`

Linha de erro relevante:

    App.java:4: error: package com.fasterxml.jackson.databind does not exist
    import com.fasterxml.jackson.databind.ObjectMapper;

A compilação falhou (`> Task :compileJava FAILED`, 5 erros) porque o `App.java` usa
`ObjectMapper` e `TypeReference` do Jackson, mas o `build.gradle` não declara a dependência.
**Dependência em falta: `com.fasterxml.jackson.core:jackson-databind`.**

## Evidence 8.2

Adicionei `implementation 'com.fasterxml.jackson.core:jackson-databind:2.22.2'` e corri
`gradle clean build`: compilou com sucesso. Depois executei
`gradle dependencies --configuration runtimeClasspath`.

`jackson-databind` é a dependência direta; `jackson-core` e `jackson-annotations` são
transitivas, trazidas pelo `jackson-databind`.

**Comparação com `mvn dependency:tree`:** as bibliotecas e as versões são as mesmas
(`jackson-databind` 2.22.2, `jackson-core` 2.22.2, `jackson-annotations` 2.22).
Mudar de build system não mudou as dependências da aplicação, só a forma de as declarar
e de as listar. O Gradle mostra ainda o `jackson-bom`, usado como restrição de versões
(marcado com `(c)`), e o Maven mostra o JUnit com escopo `test`, que a configuração
`runtimeClasspath` não inclui.

## Evidence 8.3

**JAR por omissão** (`java -jar build/libs/fleetcheck-1.0.0.jar`):

    no main manifest attribute, in build\libs\fleetcheck-1.0.0.jar

**Depois de configurar o `jar`:**

    FleetCheck 1.0 | Vehicles loaded: 4 | Vehicles requiring service: 2 | Average mileage: 37000 km

**O que mudou no JAR:** o JAR por omissão só tem as classes do projeto e não declara a
`Main-Class`, por isso não corre sozinho. Com a configuração, o manifesto passou a ter
`Main-Class: pt.upt.fleetcheck.App` e o bloco `from { ... zipTree(it) }` copiou para dentro
do JAR o conteúdo das dependências de execução (Jackson). O resultado é um JAR autónomo,
equivalente ao `-all.jar` do Shade no Maven.

