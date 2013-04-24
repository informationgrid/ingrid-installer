UPDATE-Modifikationen
=====================

(ANTINSTALL-CONFIG.XML)
-----------------------
1) Erstelle Page mit Auswahl "Update" oder "Neu"

        <target-select
            property      ="updateOrNew"
            displayText   ="Neuinstallation oder Update"
            defaultValue  ="true">
            <option value="setUpdateProperty" text="Update (Einstellungen bleiben erhalten)"/>
            <option value="setInstallProperty" text="Neuinstallation"/>
        </target-select>

2) Erstelle Page, die nur bei Update angezeigt wird. Eingabe des Zielverzeichnisses.
   
    Wichtige Attribute der Page:
        target               ="patchFiles"
        postDisplayTarget    ="antinstaller-determineVersion"

(BUILD.XML)
-----------
3) Definition der folgenden Properties:

        <!-- THIS PORPERTIES ARE NECESSARY FOR UPDATES -->
        <property name="libraryDir"         value="lib" />
        <property name="libraryIdent"       value="ingrid-iplug-csw-dsc-" />
        <!-- <property name="libraryIdent"       value="MANIFEST.MF" /> -->
        <property name="versionsWithConfigurations"  value="3.2.0,3.2.1,3.3.0" />
        <property name="minSupportedVersion" value="3.2.0" />

        libraryDir: zeigt auf das Verzeichnis wo sich die Jar oder Properties-Datei befindet
        libraryIdent: bestimmt den Dateinamen, wo sich die Manifest-Datei befindet oder ist diese selbst
        versionsWithConfigurations: enthält die Versionen, die im Antinstaller zusätzlich konfiguriert werden müssen
            Dies muss nicht für unkonfigurierte Patches geschehen!
        minSupportedVersion: ab wann wird das Update (mit ihren Patches) unterstützt

3.1) Basedir muss im Projekt als Attribut gesetzt sein!

        basedir="."

4) Importieren von externen build-Skripten:

        <import file="build-installer-utils.xml"  as="utils" />
        <import file="build-patch.xml"            as="patch" />

    Ersteres enthält nützliche Funktionen die vom Installer bereitgestellt werden.
    Letzteres enthält die speziellen Patch-Aufrufe für die Komponente. 

5) Änderungen am Installationsablauf durch:

        <target name="setUpdateProperty" depends="checkPreconditionsForUpdate, extractUpdate">
            <property name="installType" value="update" />
            <property name="updateProcedure" value="true" />
        </target>
        
        <target name="setInstallProperty" depends="extract">
            <property name="installProcedure" value="true" />
        </target>

    Durch "depends" wird das spezielle Target vorher ausgeführt. Wichtig hierbei, dass beim Update keine Konfigurations-Dateien entpackt werden! Das spezielle Target "checkPreconditionsForUpdate" überprüft, ob alle Vorbedingungen korrekt sind und schlägt ansonsten fehl!

6) Das Target "extractUpdate" (Beispiel):

        <target name="extractUpdate">
            <unzip src="${antinstaller.jar}" dest=".">
                <patternset>
                    <include name="**/*.*"/>
                    <exclude name="${componentName}/webapp/WEB-INF/spring.xml" />
                </patternset>
            </unzip>
            
            <delete>
                <fileset dir="${installDir}/lib" includes="**/*"/>
            </delete>
            
            <move toDir="${installDir}">
                <fileset dir="./${componentName}"/>
            </move>
        </target>

    Zuerst wird die Komponente im Temp-Verzeichnis ohne speziell definierte Konfigdateien entpackt.
    Dann wird das lib-Verzeichnis in der alten Installation gelöscht, um Konflikte zu vermeiden.
    Zum Schluss werden die neuen Dateien an die richtige Stelle im alten Installationspfad verschoben.

(BUILD-PATCH.XML)
-----------------
7)  Add target to build-patch.xml:

    <project name="<component-name> Patches">
        <target name="patchFiles" depends="">
            <!-- patch order determined through depends-order -->
        </target>
    </project>

    In dem depends-Attribute werden die Targets in der Reihenfolge der Ausführung eingetragen, beginnend mit dem frühesten Patch.
    Jedes Patch-Target prüft selbst, ob es ausgeführt werden muss!

8) Add a Patch-Target if needed:

        <target name="patchFromVersion3.2.1.1">
            <compareVersion value1="${oldVersion}" value2="3.2.1.1" prop="compResult"/>
            <if>
                <not>
                    <equals arg1="${compResult}" arg2="1" />
                </not>
                <then>
                    <echo>Patching spring.xml file in basedir: </echo>
                    <patchFile patchFile="patches/3.3.0/spring.xml.patch" originalFile="${installDir}/webapp/WEB-INF/spring.xml" />
                </then>
            </if>
        </target>

    Die Patches sollten im ant-installer-Verzeichnis abgelegt werden in folgender Struktur:

        ant-installer
        |--patches
           |--3.2.0
              |--spring.xml.patch
           |--3.2.1
              |--spring.xml.patch
           |--3.3.0
              |--communication.xml.patch