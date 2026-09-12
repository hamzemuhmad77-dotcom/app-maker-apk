name: Build APK

on:
  workflow_dispatch:
    inputs:
      gist_id:
        description: "Gist ID"
        required: true
        type: string

      gist_owner:
        description: "Gist owner"
        required: true
        type: string

      app_name:
        description: "Application name"
        required: true
        type: string

      package_name:
        description: "Android package name"
        required: true
        type: string

      version:
        description: "Application version"
        required: true
        default: "1.0.0"
        type: string

      version_code:
        description: "Android version code"
        required: true
        default: "1"
        type: string

      build_id:
        description: "Unique build ID"
        required: true
        type: string


permissions:
  contents: write


jobs:

  build:

    runs-on: ubuntu-latest

    steps:

      # -----------------------------------------
      # 1. تحميل المستودع
      # -----------------------------------------

      - name: Checkout repository
        uses: actions/checkout@v4


      # -----------------------------------------
      # 2. تثبيت Flutter
      # -----------------------------------------

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true


      # -----------------------------------------
      # 3. تجهيز الأدوات
      # -----------------------------------------

      - name: Install tools
        run: |
          sudo apt-get update
          sudo apt-get install -y jq


      # -----------------------------------------
      # 4. إنشاء مجلد البناء
      # -----------------------------------------

      - name: Create build directory
        run: |
          rm -rf build_app
          mkdir -p build_app


      # -----------------------------------------
      # 5. تنزيل ملفات التطبيق من Gist
      # -----------------------------------------

      - name: Download application files
        env:
          GIST_ID: ${{ inputs.gist_id }}
          GIST_OWNER: ${{ inputs.gist_owner }}
        run: |

          set -e

          curl -L \
            "https://gist.githubusercontent.com/${GIST_OWNER}/${GIST_ID}/raw/main.dart" \
            -o build_app/main.dart

          curl -L \
            "https://gist.githubusercontent.com/${GIST_OWNER}/${GIST_ID}/raw/app.json" \
            -o build_app/app.json

          curl -L \
            "https://gist.githubusercontent.com/${GIST_OWNER}/${GIST_ID}/raw/icon.b64" \
            -o build_app/icon.b64

          echo "Downloaded files:"
          ls -lah build_app


      # -----------------------------------------
      # 6. التحقق من البيانات
      # -----------------------------------------

      - name: Validate application data
        env:
          PACKAGE_NAME: ${{ inputs.package_name }}
        run: |

          set -e

          if [ ! -s "build_app/main.dart" ]; then
            echo "Flutter code is empty."
            exit 1
          fi

          if ! echo "$PACKAGE_NAME" | grep -Eq \
            '^[a-zA-Z][a-zA-Z0-9_]*(\.[a-zA-Z][a-zA-Z0-9_]*)+$'
          then
            echo "Invalid Android package name."
            exit 1
          fi

          echo "Application data is valid."


      # -----------------------------------------
      # 7. استخراج org واسم المشروع
      # -----------------------------------------

      - name: Prepare Flutter project names
        id: names
        env:
          PACKAGE_NAME: ${{ inputs.package_name }}
        run: |

          set -e

          PROJECT_NAME=$(echo "$PACKAGE_NAME" | awk -F. '{print $NF}')

          ORG=$(echo "$PACKAGE_NAME" | sed "s/\\.${PROJECT_NAME}$//")

          echo "project_name=$PROJECT_NAME" >> "$GITHUB_OUTPUT"
          echo "org=$ORG" >> "$GITHUB_OUTPUT"

          echo "Project name: $PROJECT_NAME"
          echo "Organization: $ORG"


      # -----------------------------------------
      # 8. إنشاء مشروع Flutter جديد
      # -----------------------------------------

      - name: Create Flutter project
        env:
          PROJECT_NAME: ${{ steps.names.outputs.project_name }}
          ORG: ${{ steps.names.outputs.org }}
        run: |

          set -e

          flutter create \
            --empty \
            --platforms=android \
            --org "$ORG" \
            --project-name "$PROJECT_NAME" \
            build_app/flutter_project


      # -----------------------------------------
      # 9. وضع main.dart
      # -----------------------------------------

      - name: Install application code
        run: |

          set -e

          cp build_app/main.dart \
            build_app/flutter_project/lib/main.dart


      # -----------------------------------------
      # 10. تعديل اسم التطبيق
      # -----------------------------------------

      - name: Configure application name
        env:
          APP_NAME: ${{ inputs.app_name }}
        run: |

          set -e

          python3 <<'PY'
          import os
          from pathlib import Path

          app_name = os.environ["APP_NAME"]

          manifest = Path(
              "build_app/flutter_project/android/app/src/main/AndroidManifest.xml"
          )

          text = manifest.read_text()

          text = text.replace(
              'android:label="build_app"',
              f'android:label="{app_name}"'
          )

          text = text.replace(
              'android:label="${applicationName}"',
              f'android:label="{app_name}"'
          )

          manifest.write_text(text)
          PY


      # -----------------------------------------
      # 11. ضبط الإصدار
      # -----------------------------------------

      - name: Configure application version
        env:
          VERSION: ${{ inputs.version }}
          VERSION_CODE: ${{ inputs.version_code }}
        run: |

          set -e

          python3 <<'PY'
          import os
          from pathlib import Path
          import re

          version = os.environ["VERSION"]
          version_code = os.environ["VERSION_CODE"]

          pubspec = Path(
              "build_app/flutter_project/pubspec.yaml"
          )

          text = pubspec.read_text()

          text = re.sub(
              r'^version:.*$',
              f'version: {version}+{version_code}',
              text,
              flags=re.MULTILINE
          )

          pubspec.write_text(text)
          PY


      # -----------------------------------------
      # 12. التأكد من Package Name
      # -----------------------------------------

      - name: Configure Android package
        env:
          PACKAGE_NAME: ${{ inputs.package_name }}
        run: |

          set -e

          python3 <<'PY'
          import os
          from pathlib import Path
          import re

          package = os.environ["PACKAGE_NAME"]

          gradle = Path(
              "build_app/flutter_project/android/app/build.gradle.kts"
          )

          text = gradle.read_text()

          text = re.sub(
              r'applicationId\s*=\s*"[^"]+"',
              f'applicationId = "{package}"',
              text
          )

          gradle.write_text(text)
          PY


      # -----------------------------------------
      # 13. تجهيز أيقونة التطبيق
      # -----------------------------------------

      - name: Prepare application icon
        run: |

          set -e

          ICON_FILE="build_app/icon.b64"

          if [ -s "$ICON_FILE" ]; then

            echo "Icon detected."

            mkdir -p \
              build_app/flutter_project/assets/icon

            base64 -d "$ICON_FILE" \
              > build_app/flutter_project/assets/icon/icon.png

            file \
              build_app/flutter_project/assets/icon/icon.png

          else

            echo "No custom icon supplied."

          fi


      # -----------------------------------------
      # 14. إضافة flutter_launcher_icons عند وجود أيقونة
      # -----------------------------------------

      - name: Configure launcher icon
        run: |

          set -e

          if [ -f "build_app/flutter_project/assets/icon/icon.png" ]; then

            cd build_app/flutter_project

            cat >> pubspec.yaml <<'EOF'

          dev_dependencies:
            flutter_launcher_icons: ^0.14.4

          flutter_launcher_icons:
            android: true
            ios: false
            image_path: assets/icon/icon.png
            adaptive_icon_background: "#FFFFFF"
            adaptive_icon_foreground: assets/icon/icon.png
          EOF

            flutter pub get

            dart run flutter_launcher_icons

          else

            echo "Skipping launcher icon."

          fi


      # -----------------------------------------
      # 15. تثبيت حزم Flutter
      # -----------------------------------------

      - name: Flutter pub get
        working-directory: build_app/flutter_project
        run: |

          flutter pub get


      # -----------------------------------------
      # 16. إعداد توقيع APK
      # -----------------------------------------

      - name: Configure release signing
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
          KEY_STORE_PASSWORD: ${{ secrets.KEY_STORE_PASSWORD }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
        run: |

          set -e

          if [ -z "$KEYSTORE_BASE64" ]; then
            echo "KEYSTORE_BASE64 secret is missing."
            exit 1
          fi

          if [ -z "$KEY_STORE_PASSWORD" ]; then
            echo "KEY_STORE_PASSWORD secret is missing."
            exit 1
          fi

          if [ -z "$KEY_PASSWORD" ]; then
            echo "KEY_PASSWORD secret is missing."
            exit 1
          fi

          if [ -z "$KEY_ALIAS" ]; then
            echo "KEY_ALIAS secret is missing."
            exit 1
          fi

          echo "$KEYSTORE_BASE64" | base64 --decode \
            > build_app/flutter_project/android/app/upload-keystore.jks

          cat > build_app/flutter_project/android/key.properties <<EOF
          storePassword=$KEY_STORE_PASSWORD
          keyPassword=$KEY_PASSWORD
          keyAlias=$KEY_ALIAS
          storeFile=upload-keystore.jks
          EOF


      # -----------------------------------------
      # 17. ربط توقيع Release مع Gradle
      # -----------------------------------------

      - name: Apply release signing configuration
        run: |

          set -e

          python3 <<'PY'
          from pathlib import Path

          gradle = Path(
              "build_app/flutter_project/android/app/build.gradle.kts"
          )

          text = gradle.read_text()

          signing_code = r'''

          import java.util.Properties
          import java.io.FileInputStream

          val keystorePropertiesFile =
              rootProject.file("key.properties")

          val keystoreProperties = Properties()

          if (keystorePropertiesFile.exists()) {
              keystoreProperties.load(
                  FileInputStream(keystorePropertiesFile)
              )
          }

          '''

          if "keystorePropertiesFile" not in text:
              text = signing_code + text

          if "signingConfigs {" not in text:

              marker = "android {"

              signing_block = r'''
              signingConfigs {
                  create("release") {
                      keyAlias = keystoreProperties["keyAlias"] as String
                      keyPassword = keystoreProperties["keyPassword"] as String
                      storeFile = file(
                          keystoreProperties["storeFile"] as String
                      )
                      storePassword =
                          keystoreProperties["storePassword"] as String
                  }
              }

              '''

              text = text.replace(
                  marker,
                  marker + "\n" + signing_block,
                  1
              )

          text = text.replace(
              'signingConfig = signingConfigs.getByName("debug")',
              'signingConfig = signingConfigs.getByName("release")'
          )

          text = text.replace(
              'signingConfig = signingConfigs.debug',
              'signingConfig = signingConfigs.getByName("release")'
          )

          if 'signingConfig = signingConfigs.getByName("release")' not in text:

              marker = "buildTypes {"

              release_block = r'''
              buildTypes {
                  getByName("release") {
                      signingConfig =
                          signingConfigs.getByName("release")
                  }

              '''

              text = text.replace(
                  marker,
                  release_block,
                  1
              )

          gradle.write_text(text)
          PY


      # -----------------------------------------
      # 18. فحص Flutter
      # -----------------------------------------

      - name: Flutter analyze
        working-directory: build_app/flutter_project
        run: |

          flutter analyze


      # -----------------------------------------
      # 19. بناء APK Release
      # -----------------------------------------

      - name: Build APK
        working-directory: build_app/flutter_project
        env:
          VERSION: ${{ inputs.version }}
          VERSION_CODE: ${{ inputs.version_code }}
        run: |

          set -e

          flutter build apk \
            --release \
            --build-name="$VERSION" \
            --build-number="$VERSION_CODE"


      # -----------------------------------------
      # 20. تجهيز اسم APK
      # -----------------------------------------

      - name: Prepare APK
        env:
          APP_NAME: ${{ inputs.app_name }}
          BUILD_ID: ${{ inputs.build_id }}
        run: |

          set -e

          APK_SOURCE="build_app/flutter_project/build/app/outputs/flutter-apk/app-release.apk"

          if [ ! -f "$APK_SOURCE" ]; then
            echo "APK was not created."
            exit 1
          fi

          mkdir -p release

          SAFE_NAME=$(echo "$APP_NAME" | \
            tr '[:upper:]' '[:lower:]' | \
            tr -cd 'a-zA-Z0-9_-')

          if [ -z "$SAFE_NAME" ]; then
            SAFE_NAME="app"
          fi

          cp "$APK_SOURCE" \
            "release/${SAFE_NAME}-${BUILD_ID}.apk"

          echo "APK created:"
          ls -lh release/


      # -----------------------------------------
      # 21. إنشاء GitHub Release
      # -----------------------------------------

      - name: Publish APK release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BUILD_ID: ${{ inputs.build_id }}
          APP_NAME: ${{ inputs.app_name }}
          VERSION: ${{ inputs.version }}
        run: |

          set -e

          APK_FILE=$(find release -name "*.apk" | head -n 1)

          if [ -z "$APK_FILE" ]; then
            echo "APK file not found."
            exit 1
          fi

          gh release create \
            "builder-${BUILD_ID}" \
            "$APK_FILE" \
            --repo "${GITHUB_REPOSITORY}" \
            --title "${APP_NAME} v${VERSION}" \
            --notes "APK generated by App Builder."


      # -----------------------------------------
      # 22. نجاح البناء
      # -----------------------------------------

      - name: Build completed
        run: |

          echo "===================================="
          echo "APK BUILD COMPLETED SUCCESSFULLY"
          echo "====================================" 
