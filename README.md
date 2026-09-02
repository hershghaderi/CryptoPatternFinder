import zipfile, os, shutil, re, json

src="/mnt/data/project/CryptoPatternFinder_Complete-2.zip"
work="/mnt/data/CryptoPatternFinder_fixed"
out="/mnt/data/CryptoPatternFinder_Complete_FIXED.zip"

if os.path.exists(work):
    shutil.rmtree(work)
os.makedirs(work)

with zipfile.ZipFile(src, "r") as z:
    z.extractall(work)

# Find Gradle Kotlin build files and normalize Java/Kotlin settings.
for root, dirs, files in os.walk(work):
    for fn in files:
        if fn.endswith((".gradle.kts", ".gradle", ".properties", ".yml", ".yaml")):
            p=os.path.join(root,fn)
            try:
                s=open(p,encoding="utf-8").read()
            except:
                continue
            # Fix obsolete Java 8 toolchain references where present.
            s=re.sub(r'JavaLanguageVersion\.of\(\s*8\s*\)', 'JavaLanguageVersion.of(17)', s)
            s=re.sub(r'jvmTarget\s*=\s*["\']8["\']', 'jvmTarget = "17"', s)
            s=re.sub(r'jvmToolchain\(\s*8\s*\)', 'jvmToolchain(17)', s)
            s=re.sub(r'languageVersion\s*=\s*JavaLanguageVersion\.of\(\s*8\s*\)', 'languageVersion = JavaLanguageVersion.of(17)', s)
            # GitHub Actions: use JDK 17 consistently.
            if fn.endswith((".yml",".yaml")):
                s=s.replace("java-version: '8'", "java-version: '17'")
                s=s.replace('java-version: "8"', 'java-version: "17"')
                s=s.replace("java-version: 8", "java-version: 17")
                s=s.replace("actions/setup-java@v4", "actions/setup-java@v5")
            open(p,"w",encoding="utf-8").write(s)

# Fix common Android/Kotlin DSL issue: kotlinOptions block can fail with newer plugin;
# replace simple kotlinOptions { jvmTarget = "17" } with compilerOptions when possible.
for root, dirs, files in os.walk(work):
    for fn in files:
        if fn.endswith(".gradle.kts"):
            p=os.path.join(root,fn)
            s=open(p,encoding="utf-8").read()
            if "kotlinOptions" in s and "compilerOptions" not in s:
                s=s.replace(
                    'kotlinOptions {\n        jvmTarget = "17"\n    }',
                    'compilerOptions {\n        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)\n    }'
                )
                s=s.replace(
                    'kotlinOptions {\n            jvmTarget = "17"\n        }',
                    'compilerOptions {\n            jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)\n        }'
                )
            open(p,"w",encoding="utf-8").write(s)

# Remove nested zip(s) that could confuse uploads, while preserving project source.
for root, dirs, files in os.walk(work):
    for fn in files:
        if fn.lower().endswith(".zip") and os.path.join(root,fn) != out:
            os.remove(os.path.join(root,fn))

if os.path.exists(out):
    os.remove(out)
with zipfile.ZipFile(out, "w", zipfile.ZIP_DEFLATED) as z:
    for root, dirs, files in os.walk(work):
        for fn in files:
            path=os.path.join(root,fn)
            arc=os.path.relpath(path, work)
            z.write(path, arc)

print(f"[دانلود فایل اصلاح‌شده](sandbox:{out})")
