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

---

# Relatório

**Unidade Curricular:** Software Quality, Universidade Portucalense
**Aluno:** Guilherme da Costa Ribeiro, n.º 52382

## Passo 1: Build sem alterações

**Comando executado:** `mvn clean package`

**Resultado:** `BUILD FAILURE`, com 5 erros de compilação, todos em `App.java`.

**Evidence 1:**

```
[ERROR] /C:/Users/gcost/Downloads/FleetCheck_Starter/src/main/java/pt/upt/fleetcheck/App.java:[4,38] package com.fasterxml.jackson.databind does not exist
```

**Análise:** O build falhou na fase de compilação (`maven-compiler-plugin:3.16.0:compile`). A origem está nos imports das linhas 3 e 4 de `App.java`:

- `com.fasterxml.jackson.core.type.TypeReference` (linha 3)
- `com.fasterxml.jackson.databind.ObjectMapper` (linha 4)

O `pom.xml` não declara a dependência `jackson-databind`, pelo que o Maven não a coloca no classpath de compilação. O compilador não encontra os pacotes dos imports e, consequentemente, também não encontra as classes `ObjectMapper` (linha 11) e `TypeReference` (linha 20). O problema está na configuração do build e não no código-fonte.

## Passo 2: Dependência explícita

Foi adicionada ao `pom.xml`, antes da dependência do JUnit:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.22.2</version>
</dependency>
```

**Comando executado:** `mvn clean package`

**Resultado:** o Maven descarregou `jackson-databind-2.22.2` e as suas dependências (`jackson-core-2.22.2` e `jackson-annotations-2.22`) do Maven Central. A compilação passou e o build terminou em `BUILD SUCCESS`, com a geração de `target/fleetcheck-1.0.0.jar`.

**Observação:** a ficha previa que a fase de testes falhasse, mas nesta execução não foram executados testes: a pasta `src/test` existe no projeto fornecido mas não contém nenhum teste, pelo que o log do `surefire:test` não mostra `Tests run: ...`. Por indicação da professora, este ponto foi ignorado e o trabalho seguiu para o passo seguinte.

**Pergunta: porque é esta falha melhor do que a do Passo 1?**

A falha do Passo 1 era um problema de configuração do build: sem a dependência declarada, o código nem sequer compilava, por isso nada sobre o comportamento da aplicação podia ser verificado. Depois de declarar a dependência de forma explícita, o build passa a fase de compilação e chega à fase de testes, que é onde se verifica o comportamento do código. Uma falha nessa fase indicaria um defeito real da aplicação, e não uma falta de configuração. O build avançou assim mais no ciclo de vida do Maven (`compile` → `test` → `package`), e é esse avanço que torna o resultado melhor.

## Passo 3: Árvore de dependências

**Comando executado:** `mvn dependency:tree`

**Resultado (parte relevante):**

```
[INFO] pt.upt.softwarequality:fleetcheck:jar:1.0.0
[INFO] +- com.fasterxml.jackson.core:jackson-databind:jar:2.22.2:compile
[INFO] |  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.22:compile
[INFO] |  \- com.fasterxml.jackson.core:jackson-core:jar:2.22.2:compile
[INFO] \- org.junit.jupiter:junit-jupiter:jar:5.14.4:test
[INFO]    +- org.junit.jupiter:junit-jupiter-api:jar:5.14.4:test
[INFO]    |  +- org.opentest4j:opentest4j:jar:1.3.0:test
[INFO]    |  +- org.junit.platform:junit-platform-commons:jar:1.14.4:test
[INFO]    |  \- org.apiguardian:apiguardian-api:jar:1.1.2:test
[INFO]    +- org.junit.jupiter:junit-jupiter-params:jar:5.14.4:test
[INFO]    \- org.junit.jupiter:junit-jupiter-engine:jar:5.14.4:test
[INFO]       \- org.junit.platform:junit-platform-engine:jar:1.14.4:test
[INFO] BUILD SUCCESS
```

**Análise:**

- `jackson-databind:2.22.2` é uma dependência **direta**, porque está declarada no `pom.xml`.
- `jackson-core:2.22.2` e `jackson-annotations:2.22` são dependências **transitivas**: não foram declaradas por mim, mas o Maven resolveu-as automaticamente porque o `jackson-databind` depende delas (aparecem indentadas por baixo dele na árvore).
- Todas têm scope `compile`, ou seja, estão disponíveis na compilação e em tempo de execução. O JUnit e as suas dependências têm scope `test`, pelo que só são usados na fase de testes.

## Passo 4: JAR executável (Shade)

**JAR normal (sem Shade):**

```
PS> java -jar target/fleetcheck-1.0.0.jar
no main manifest attribute, in target/fleetcheck-1.0.0.jar
```

**JAR com Shade (após adicionar o `maven-shade-plugin`):** na primeira execução, a aplicação arrancou mas o resultado era incorreto:

```
PS> java -jar target/fleetcheck-1.0.0-all.jar
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 1
Average mileage: 37000 km
```

**Defeito encontrado e corrigido:** o valor esperado era `2`. A causa estava em `FleetService.needsService`, que usava `>` na comparação entre os quilómetros desde a última revisão e o intervalo de serviço. O veículo V3 (65000 − 55000 = 10000 km, com intervalo de 10000 km) está exatamente no limite e não era contado. Foi alterado para `>=`:

```java
// antes
return kilometresSinceService > vehicle.serviceIntervalKm();
// depois
return kilometresSinceService >= vehicle.serviceIntervalKm();
```

**Resultado após a correção:**

```
PS> mvn clean package
...
[INFO] --- shade:3.6.2:shade (default) @ fleetcheck ---
[INFO] Attaching shaded artifact.
[INFO] BUILD SUCCESS

PS> java -jar target/fleetcheck-1.0.0-all.jar
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

**Conteúdo de `target/` (antes da correção):**

```
2377613  fleetcheck-1.0.0-all.jar
   6425  fleetcheck-1.0.0.jar
```

**Evidence 4: o que mudou o plugin Shade em relação ao JAR por omissão?**

O JAR por omissão (`fleetcheck-1.0.0.jar`, cerca de 6 KB) contém apenas as classes compiladas e os recursos do projeto. O seu manifesto não tem o atributo `Main-Class`, por isso o `java -jar` não sabe que classe arrancar e falha com `no main manifest attribute`. Além disso, não inclui as bibliotecas de que a aplicação precisa em tempo de execução (Jackson), que ficam apenas no repositório local do Maven.

O Shade gerou um segundo artefacto, `fleetcheck-1.0.0-all.jar` (cerca de 2,3 MB), com duas diferenças:

1. **Manifesto:** o `ManifestResourceTransformer` escreveu `Main-Class: pt.upt.fleetcheck.App`, pelo que o JAR passa a ser executável.
2. **Dependências embutidas:** as classes de `jackson-databind`, `jackson-core` e `jackson-annotations` foram copiadas para dentro do JAR ("fat JAR" ou "uber JAR"), que passa a ser autossuficiente e pode ser executado noutra máquina sem Maven nem classpath externo.

A diferença de tamanho (6 KB contra 2,3 MB) deve-se às bibliotecas Jackson embutidas. Como `shadedArtifactAttached` está a `true`, o JAR original foi mantido e o novo foi acrescentado com o classificador `all`.

**Nota sobre os avisos do Shade:** durante o build surgiram `[WARNING]` sobre ficheiros repetidos em vários JARs (`META-INF/LICENSE`, `META-INF/NOTICE`, `META-INF/MANIFEST.MF` e `module-info`). Quando vários JARs contêm o mesmo ficheiro, o Shade copia apenas uma versão para o JAR final. Estes avisos não afetam o funcionamento da aplicação.

## Passo 5: Maven Wrapper

**Comando executado:** `mvn wrapper:wrapper`

**Resultado:** o plugin `maven-wrapper-plugin:3.3.4` gerou os ficheiros `mvnw`, `mvnw.cmd` e `.mvn/wrapper/maven-wrapper.properties`, configurado para usar o Maven 3.10.0 a partir do Maven Central:

```
[INFO] Configuring .mvn/wrapper/maven-wrapper.properties to use Maven 3.10.0 and download from https://repo.maven.apache.org/maven2
[INFO] BUILD SUCCESS
```

Foi também acrescentada ao `pom.xml` a propriedade `project.build.outputTimestamp` (`2026-09-27T00:00:00Z`), que fixa a data de criação dos ficheiros dentro do JAR, para que o mesmo código produza o mesmo artefacto. Os ficheiros do wrapper foram adicionados ao Git. A partir daqui o build é executado com o wrapper:

```
PS> .\mvnw.cmd clean verify
...
[INFO] BUILD SUCCESS
```

**Pergunta: que pressuposto ambiental removeu o wrapper?**

Removeu o pressuposto de que cada máquina tem o Maven instalado, na versão certa e configurado no `Path`. Sem wrapper, o resultado do build depende da versão de Maven que cada programador (ou o servidor de CI) tiver instalada, e quem não tiver Maven não consegue compilar. Com o wrapper, o projeto indica no `maven-wrapper.properties` a versão exata (3.10.0), e o `mvnw` descarrega-a automaticamente na primeira execução. Assim, qualquer máquina usa a mesma versão do Maven, o que torna o build repetível. O wrapper continua a pressupor que existe um JDK disponível.

## Passo 6: GitHub Actions (Maven)

Foi criado o repositório `worksheet-build-systems` no GitHub, com o projeto e o ficheiro `.github/workflows/build.yml` (workflow "Build and Verify"). O workflow faz checkout, instala o Java 21 (Temurin, com cache Maven), executa `./mvnw -B clean verify` e carrega o artefacto de build.

**Resultado:** a execução terminou com sucesso (`Success`, 19 s) e tem 1 artefacto anexado, `fleetcheck-build`.

**URL da execução:** https://github.com/guiribeiro50/worksheet-build-systems/actions/runs/37118854508

**Nota:** a execução mostrou um aviso informativo do GitHub a indicar que o rótulo `ubuntu-latest` passará a apontar para o Ubuntu 26 a partir de 19 de outubro de 2026. Não afetou o resultado.

## Passo 7: SBOM (Maven)

Foram acrescentados ao `pom.xml` a propriedade `cyclonedx.version` (`2.9.3`) e o plugin `cyclonedx-maven-plugin`, ligado à fase `verify` com o objetivo `makeAggregateBom`, formato JSON e nome `bom`.

**Comando executado:** `.\mvnw.cmd clean verify`

**Resultado:** `BUILD SUCCESS`. O log do plugin indica:

```
[INFO] --- cyclonedx:2.9.3:makeAggregateBom (default) @ fleetcheck ---
[INFO] CycloneDX: Creating BOM version 1.6 with 3 component(s)
[INFO] CycloneDX: Writing and validating BOM (JSON): ...\target\bom.json
```

O ficheiro `target/bom.json` foi criado (10839 bytes). A pesquisa por `jackson` no ficheiro encontra os componentes:

```
target\bom.json:84:  "bom-ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.22.2?type=jar",
target\bom.json:87:  "name" : "jackson-databind",
target\bom.json:154: "bom-ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-annotations@2.22?type=jar",
```

(O `jackson-core@2.22.2` é o terceiro componente do BOM.)

**Evidence 7: porque é que o SBOM contém componentes que não escrevi na secção `dependencies` original?**

Porque o SBOM (Software Bill of Materials) descreve tudo o que faz parte do software, e não apenas o que foi declarado à mão. No `pom.xml` só declarei o `jackson-databind`, mas este depende, por sua vez, do `jackson-core` e do `jackson-annotations`. O Maven resolve essas dependências transitivas automaticamente (como se viu no `mvn dependency:tree`), e o plugin CycloneDX lê o grafo de dependências já resolvido, por isso o BOM tem 3 componentes em vez de 1. Isto é o comportamento pretendido: se existir uma vulnerabilidade conhecida no `jackson-core`, por exemplo, só é possível detetá-la se esse componente constar do inventário, mesmo que nunca tenha sido escrito no `pom.xml`. O JUnit não aparece porque tem scope `test` e não faz parte do artefacto entregue.

## Passo 8: Gradle

### 8.1 Build sem Jackson

Foi feita uma cópia do projeto Maven para `FleetCheck_Gradle`, mantendo a pasta `src/` completa (já com a correção `>=`). Foram removidos o `pom.xml` e o wrapper do Maven (`mvnw`, `mvnw.cmd`) e criados o `settings.gradle` (`rootProject.name = 'fleetcheck'`) e o `build.gradle` inicial, com o plugin `java`, toolchain Java 21 e apenas o JUnit como dependência de teste (o Jackson ficou de fora de propósito).

**Comando executado:** `gradle clean build`

**Resultado:** `BUILD FAILED in 8s`, na tarefa `:compileJava`, com 5 erros de compilação em `App.java`.

**Evidence 8.1: linha de erro relevante:**

```
C:\Users\gcost\Downloads\FleetCheck_Gradle\src\main\java\pt\upt\fleetcheck\App.java:4: error: package com.fasterxml.jackson.databind does not exist
import com.fasterxml.jackson.databind.ObjectMapper;
```

**Dependência em falta:** `com.fasterxml.jackson.core:jackson-databind`. Os imports das linhas 3 e 4 de `App.java` (`TypeReference` e `ObjectMapper`) pertencem ao Jackson, e o `build.gradle` não o declara, pelo que o Gradle não o coloca no classpath de compilação. É o mesmo erro que o Maven deu no Passo 1, o que mostra que o problema está na falta da dependência e não na ferramenta de build.

### 8.2 Dependências

Foi acrescentada ao bloco `dependencies` do `build.gradle` a linha:

```groovy
implementation 'com.fasterxml.jackson.core:jackson-databind:2.22.2'
```

**Comando executado:** `gradle clean build`

**Resultado:** `BUILD SUCCESSFUL in 3s`. A aplicação compila. Tal como no Maven, não foram executados testes, porque a pasta `src/test` está vazia.

**Comando executado:** `gradle dependencies --configuration runtimeClasspath`

**Resultado:**

```
runtimeClasspath - Runtime classpath of source set 'main'.
\--- com.fasterxml.jackson.core:jackson-databind:2.22.2
     +--- com.fasterxml.jackson.core:jackson-annotations:2.22
     +--- com.fasterxml.jackson.core:jackson-core:2.22.2
     |    \--- com.fasterxml.jackson:jackson-bom:2.22.2
     |         +--- com.fasterxml.jackson.core:jackson-annotations:2.22 (c)
     |         +--- com.fasterxml.jackson.core:jackson-core:2.22.2 (c)
     |         \--- com.fasterxml.jackson.core:jackson-databind:2.22.2 (c)
     \--- com.fasterxml.jackson:jackson-bom:2.22.2 (*)
```

**Dependências diretas e transitivas:**

- **Direta:** `jackson-databind:2.22.2`, a única declarada no `build.gradle`.
- **Transitivas:** `jackson-annotations:2.22` e `jackson-core:2.22.2`, trazidas pelo `jackson-databind`. Aparecem indentadas por baixo dele.
- O `jackson-bom:2.22.2` aparece como restrição de versões (marcado com `(c)`, "dependency constraint"), e não como biblioteca com código.

**Evidence 8.2: comparação com `mvn dependency:tree`. Mudar o sistema de build mudou as dependências da aplicação?**

Não. O Maven e o Gradle resolveram exatamente as mesmas bibliotecas, nas mesmas versões: `jackson-databind:2.22.2`, `jackson-core:2.22.2` e `jackson-annotations:2.22`, com o `databind` como única dependência direta e as outras duas como transitivas. As diferenças estão apenas na forma como cada ferramenta apresenta o resultado:

- O Maven mostra o scope (`compile`, `test`) e inclui na árvore o JUnit com scope `test`. O comando do Gradle mostra só a `runtimeClasspath`, onde o JUnit não aparece, porque é uma dependência de teste.
- O Gradle mostra o `jackson-bom` como restrição de versões `(c)`, que o Maven não apresenta na árvore.
- A palavra `implementation` do Gradle corresponde, no essencial, ao scope `compile` do Maven.

As dependências pertencem à aplicação e ao repositório (Maven Central), e não à ferramenta de build, que apenas as declara e resolve.

### 8.3 JAR executável

**JAR por omissão (`gradle clean jar`):**

```
PS> dir build\libs
4755  fleetcheck-1.0.0.jar

PS> java -jar build/libs/fleetcheck-1.0.0.jar
no main manifest attribute, in build/libs/fleetcheck-1.0.0.jar
```

Foi depois acrescentado o plugin `application` (com `mainClass = 'pt.upt.fleetcheck.App'`) e a configuração do `jar` da ficha: atributo `Main-Class` no manifesto, `duplicatesStrategy = DuplicatesStrategy.EXCLUDE` e `from { configurations.runtimeClasspath.collect { ... zipTree(it) } }`, que copia para dentro do JAR o conteúdo dos JARs da `runtimeClasspath`.

**JAR após a configuração (`gradle clean jar`):**

```
PS> dir build\libs
2356895  fleetcheck-1.0.0.jar

PS> java -jar build/libs/fleetcheck-1.0.0.jar
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

**Evidence 8.3: o que mudou no JAR depois de incluir as dependências de execução?**

O JAR passou de 4755 bytes (cerca de 5 KB) para 2356895 bytes (cerca de 2,3 MB), e passou a ser executável. Houve duas mudanças:

1. **Manifesto:** o atributo `Main-Class: pt.upt.fleetcheck.App` indica ao `java -jar` qual a classe a arrancar. Sem ele, o Java falhava com `no main manifest attribute`.
2. **Dependências embutidas:** o bloco `from { ... zipTree(it) }` abre cada JAR da `runtimeClasspath` (`jackson-databind`, `jackson-core` e `jackson-annotations`) e copia as suas classes para dentro do JAR da aplicação. O resultado é um "fat JAR", autossuficiente, que corre sem classpath externo. A diferença de tamanho deve-se a estas bibliotecas. A estratégia `EXCLUDE` evita erros quando vários JARs têm ficheiros com o mesmo nome (por exemplo, em `META-INF`): fica apenas o primeiro.

É o mesmo resultado que o Shade deu no Maven (JAR de cerca de 2,3 MB, com a mesma saída da aplicação). O que muda é o mecanismo: no Maven, um plugin dedicado (`maven-shade-plugin`), e no Gradle, a configuração da tarefa `jar`.

### 8.4 Gradle Wrapper

**Comando executado:** `gradle wrapper`

**Resultado:** `BUILD SUCCESSFUL in 1s`. Foram criados `gradlew`, `gradlew.bat` e a pasta `gradle/wrapper/`:

```
PS> dir gradlew*
8656  gradlew
4888  gradlew.bat

PS> dir gradle\wrapper
47623  gradle-wrapper.jar
  281  gradle-wrapper.properties
```

**Comando executado:** `.\gradlew.bat clean build`

**Resultado:** o wrapper descarregou a distribuição do Gradle que está fixada em `gradle-wrapper.properties` e executou o build com ela:

```
Fetching distribution.
Downloading https://services.gradle.org/distributions/gradle-9.8.0-bin.zip
...100%

BUILD SUCCESSFUL in 6s
7 actionable tasks: 7 executed
```

**Pergunta: que pressuposto escondido do ambiente removeu o Gradle Wrapper?**

Removeu o pressuposto de que o Gradle já está instalado na máquina, e na versão certa. Sem o wrapper, quem clona o projeto tem de instalar o Gradle manualmente, e uma versão diferente da usada pelo autor pode dar resultados diferentes ou até falhar. Com o wrapper, o `gradle-wrapper.properties` fixa a versão (9.8.0): o `gradlew` descarrega-a automaticamente na primeira execução (como se viu acima) e usa-a sempre. Assim, o mesmo comando produz o mesmo build em qualquer máquina, incluindo o servidor do GitHub Actions, onde não é preciso instalar o Gradle. Só é preciso ter um JDK disponível.

### 8.5 GitHub Actions (Gradle)

Foi criado o repositório `worksheet-build-systems-gradle`, com o projeto Gradle, o wrapper (`gradlew` marcado como executável com `git update-index --chmod=+x gradlew`) e o workflow `.github/workflows/build-gradle.yml`. O push para `main` acionou o workflow automaticamente.

**Resultado:** o workflow terminou em **Success**, com duração total de 1m 6s (job `build`: 26s). Foi anexado o artefacto `fleetcheck-gradle-build` (2,07 MB), que contém o JAR gerado em `build/libs/`.

**Evidence 8.5 (URL da execução com sucesso):**

https://github.com/guiribeiro50/worksheet-build-systems-gradle/actions/runs/37119980563

### 8.6 SBOM (Gradle)

Foi acrescentado ao bloco `plugins` do `build.gradle` o plugin `id 'org.cyclonedx.bom' version '3.4.1'`.

**Comando executado:** `.\gradlew.bat cyclonedxBom`

**Resultado:** `BUILD SUCCESSFUL in 13s`. O plugin gerou `build/reports/cyclonedx/bom.json` (29877 bytes) e também `bom.xml`.

A pesquisa por Jackson no `bom.json` encontra os três componentes (excerto):

```
bom.json:37:  "bom-ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-annotations@2.22?type=jar",
bom.json:100: "bom-ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-core@2.22.2?type=jar",
bom.json:163: "bom-ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.22.2?type=jar",
bom.json:804: "ref" : "pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.22.2?type=jar",
bom.json:806:   "pkg:maven/com.fasterxml.jackson.core/jackson-annotations@2.22?type=jar",
bom.json:807:   "pkg:maven/com.fasterxml.jackson.core/jackson-core@2.22.2?type=jar",
```

**Evidence 8.6: porque é que o SBOM do Gradle contém dependências que não escrevi no `build.gradle`?**

No `build.gradle` só declarei o `jackson-databind` (mais o JUnit, para testes). O SBOM, no entanto, lista também o `jackson-core` e o `jackson-annotations`. Isto acontece porque o plugin CycloneDX não lê apenas o que foi escrito à mão: lê o grafo de dependências já resolvido pelo Gradle, que inclui as dependências transitivas, ou seja, as bibliotecas de que o `jackson-databind` precisa para funcionar. A secção de dependências do `bom.json` mostra isso mesmo: o `jackson-databind` aparece com `jackson-annotations` e `jackson-core` listados como dependências suas (linhas 804 a 807).

Um SBOM serve para inventariar tudo o que compõe o software entregue, e é essa lista completa que permite, por exemplo, verificar se alguma biblioteca tem vulnerabilidades conhecidas, mesmo que nunca tenha sido declarada diretamente. O resultado é equivalente ao do Maven (Passo 7): o sistema de build mudou, mas as bibliotecas e as versões são as mesmas.

### 8.7 Comparação Maven vs Gradle

| Tarefa                   | Maven                          | Gradle                                    |
| ------------------------ | ------------------------------ | ----------------------------------------- |
| Configuração do build    | `pom.xml`                      | `build.gradle`                            |
| Build limpo              | `mvnw.cmd clean verify`        | `gradlew.bat clean build`                 |
| Adicionar dependência    | `<dependency>...</dependency>` | `implementation 'group:artifact:version'` |
| Inspecionar dependências | `mvn dependency:tree`          | `gradle dependencies`                     |
| Wrapper                  | `mvnw.cmd`                     | `gradlew.bat`                             |
| Output do build          | `target/`                      | `build/`                                  |
| Localização do JAR       | `target/`                      | `build/libs/`                             |
| SBOM                     | Plugin CycloneDX para Maven    | Plugin CycloneDX para Gradle              |

**Pergunta final: o que mudou, o software ou o processo de build?**

Mudou o processo de build, e não o software. O código-fonte (`src/`) é o mesmo nos dois projetos, as bibliotecas são as mesmas, nas mesmas versões (`jackson-databind:2.22.2`, `jackson-core:2.22.2`, `jackson-annotations:2.22`), e o JAR final produz exatamente a mesma saída:

```
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

O que mudou foi a forma de descrever e executar a construção: o ficheiro de configuração (`pom.xml` contra `build.gradle`), os comandos (`mvnw.cmd clean verify` contra `gradlew.bat clean build`), as pastas de saída (`target/` contra `build/`) e o mecanismo para gerar o JAR executável (`maven-shade-plugin` contra a configuração da tarefa `jar`). Os dois seguiram a mesma sequência de práticas de qualidade: dependências explícitas e inspecionáveis, um artefacto executável e autossuficiente, um wrapper que fixa a versão da ferramenta, integração contínua no GitHub Actions e um SBOM com as dependências transitivas.

A conclusão é que o Maven e o Gradle são duas formas de chegar ao mesmo artefacto. A qualidade do resultado depende de o build ser explícito, reprodutível e verificável, e não da ferramenta escolhida.
