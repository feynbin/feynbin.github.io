Ubuntu24.04

#### 自动配置java，gradle，maven。

#### jdk

```
#!/bin/bash
set -Eeuo pipefail

JAVA_VERSION="17"

INSTALL_DIR="$HOME/.local/share"
BASHRC_FILE="$HOME/.bashrc"

LOG_LEVEL="INFO"
LOG_COLOR_INFO='\033[1;32m'
LOG_COLOR_WARN='\033[1;33m'
LOG_COLOR_ERROR='\033[1;31m'
LOG_COLOR_DEBUG='\033[1;36m'
LOG_COLOR_RESET='\033[0m'

log() {
    local level=$1
    local message=$2
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local color=""
    case $level in
        INFO) color=${LOG_COLOR_INFO} ;;
        WARN) color=${LOG_COLOR_WARN} ;;
        ERROR) color=${LOG_COLOR_ERROR} ;;
        DEBUG) color=${LOG_COLOR_DEBUG} ;;
        *) color=${LOG_COLOR_RESET} ;;
    esac
    if [[ $level == "DEBUG" && $LOG_LEVEL != "DEBUG" ]]; then
        return
    fi
    echo -e "${color}[${timestamp}] [${level}] ${message}${LOG_COLOR_RESET}" >&2
}

die() {
    local message=$1
    local code=${2-1}
    log "ERROR" "💥 ${message}"
    exit "$code"
}

check_dependencies() {
    local dependencies=("wget" "curl" "tar")
    for dep in "${dependencies[@]}"; do
        if ! command -v "$dep" &>/dev/null; then
            die "Required tool '$dep' is not installed. Please install it and try again."
        fi
    done
    log "INFO" "✅ All required tools are installed."
}

install_java() {
    local java_url="https://mirrors.nju.edu.cn/adoptium/${JAVA_VERSION}/jdk/x64/linux"
    local java_file=$(curl -sSL "${java_url}/" | grep -oP "OpenJDK${JAVA_VERSION}U-jdk_x64_linux_hotspot_[0-9\._]+\.tar\.gz" | head -n 1)
    if [[ -z "$java_file" ]]; then
        die "Failed to find Java ${JAVA_VERSION} version."
    fi

    local java_archive="/tmp/$java_file"  # 保存到 /tmp 目录
    local java_dir="$INSTALL_DIR/jdk-${JAVA_VERSION}"

    local file_size=$(curl -sI "${java_url}/${java_file}" | grep -i 'Content-Length' | awk '{print $2}' | tr -d '\r')
    if [[ -n "$file_size" ]]; then
        file_size=$(numfmt --to=iec --suffix=B --format="%.2f" "$file_size")
        log "INFO" "🌎 Downloading JDK ${JAVA_VERSION} (${file_size})..."
    else
        log "WARN" "🌎 Downloading JDK ${JAVA_VERSION} (file size unknown)..."
    fi

    if ! wget --show-progress -q -O "$java_archive" "${java_url}/${java_file}"; then
        die "Failed to download Java"
    fi

    log "INFO" "📦 Extracting Java..."
    mkdir -p "$java_dir"
    tar -xzf "$java_archive" -C "$java_dir" --strip-components=1 || die "Failed to extract Java"
    rm -f "$java_archive"
}

configure_environment() {
    log "INFO" "🔧 Configuring Java environment variables..."

    local java_home="$INSTALL_DIR/jdk-${JAVA_VERSION}"

    if grep -q "JAVA_HOME=" "$BASHRC_FILE"; then
        sed -i "/JAVA_HOME=/c\JAVA_HOME=$java_home" "$BASHRC_FILE"
    else
        echo "JAVA_HOME=$java_home" >> "$BASHRC_FILE"
    fi

    if grep -q "PATH=" "$BASHRC_FILE"; then
        if ! grep -q "\$JAVA_HOME/bin" "$BASHRC_FILE"; then
            sed -i "s|^PATH=\(.*\)|PATH=\1:\$JAVA_HOME/bin|" "$BASHRC_FILE"
        fi
    else
        echo 'PATH=$PATH:$JAVA_HOME/bin' >> "$BASHRC_FILE"
    fi

    log "INFO" "✅ Java environment variables configured."
}

main() {
    log "INFO" "👶 Starting Java installation..."
    check_dependencies
    install_java
    configure_environment
    log "INFO" "✅ Java installation completed successfully!"
}

main
```

#### maven

```bash
#!/bin/bash
set -Eeuo pipefail

MAVEN_VERSION="4"

INSTALL_DIR="$HOME/.local/share/maven"
BASHRC_FILE="$HOME/.bashrc"

LOG_LEVEL="INFO"
LOG_COLOR_INFO='\033[1;32m'
LOG_COLOR_WARN='\033[1;33m'
LOG_COLOR_ERROR='\033[1;31m'
LOG_COLOR_DEBUG='\033[1;36m'
LOG_COLOR_RESET='\033[0m'

log() {
    local level=$1
    local message=$2
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local color=""
    case $level in
        INFO) color=${LOG_COLOR_INFO} ;;
        WARN) color=${LOG_COLOR_WARN} ;;
        ERROR) color=${LOG_COLOR_ERROR} ;;
        DEBUG) color=${LOG_COLOR_DEBUG} ;;
        *) color=${LOG_COLOR_RESET} ;;
    esac
    if [[ $level == "DEBUG" && $LOG_LEVEL != "DEBUG" ]]; then
        return
    fi
    echo -e "${color}[${timestamp}] [${level}] ${message}${LOG_COLOR_RESET}" >&2
}

die() {
    local message=$1
    local code=${2-1}
    log "ERROR" "💥 ${message}"
    exit "$code"
}

check_dependencies() {
    local dependencies=("curl" "wget" "tar")
    local missing_deps=()

    for dep in "${dependencies[@]}"; do
        if ! command -v "$dep" &>/dev/null; then
            missing_deps+=("$dep")
        fi
    done

    if [[ ${#missing_deps[@]} -gt 0 ]]; then
        log "WARN" "Missing dependencies: ${missing_deps[*]}"
        log "INFO" "Please install them and try again."
        exit 1
    fi
    log "INFO" "✅ All dependencies are installed."
}

get_maven_url() {
    local version=$1
    if [[ "$version" =~ ^3(\.[0-9]+\.[0-9]+)?$ ]]; then
        local base_version="3"
        log "DEBUG" "Fetching latest Maven 3 version..."
        local rel_version=$(curl -sSL "https://mirrors.nju.edu.cn/apache/maven/maven-${base_version}" 2>/dev/null | grep -oP 'title="[^"]*"' | sed -E 's/title="|"$//g' | tail -n 1)
        if [[ -z "$rel_version" ]]; then
            die "Failed to fetch latest Maven version"
        fi
        echo "https://mirrors.nju.edu.cn/apache/maven/maven-${base_version}/${rel_version}/binaries/apache-maven-${rel_version}-bin.tar.gz"
    elif [[ "$version" =~ ^4(\.[0-9]+\.[0-9]+)?$ ]]; then
        local base_version="4"
        log "DEBUG" "Fetching latest Maven 4 version..."
        local rel_version=$(curl -sSL "https://mirrors.nju.edu.cn/apache/maven/maven-${base_version}" 2>/dev/null | grep -oP 'title="[^"]*"' | sed -E 's/title="|"$//g' | tail -n 1)
        if [[ -z "$rel_version" ]]; then
            die "Failed to fetch latest Maven version"
        fi
        echo "https://mirrors.nju.edu.cn/apache/maven/maven-${base_version}/${rel_version}/binaries/apache-maven-${rel_version}-bin.tar.gz"
    else
        die "Unsupported MAVEN_VERSION: ${version}"
    fi
}

install_maven() {
    log "INFO" "🧹 Cleaning install directory..."
    rm -rf "${INSTALL_DIR}"
    mkdir -p "${INSTALL_DIR}"

    local maven_url=$(get_maven_url "$MAVEN_VERSION")
    local maven_file=$(basename "$maven_url")
    local tmp_file="/tmp/${maven_file}"

    local file_size=$(curl -sI "$maven_url" | grep -i 'Content-Length' | awk '{print $2}' | tr -d '\r')
    if [[ -n "$file_size" ]]; then
        file_size=$(numfmt --to=iec --suffix=B --format="%.2f" "$file_size")
        log "INFO" "Downloading ${maven_file} (${file_size})..."
    else
        log "WARN" "Downloading ${maven_file} (file size unknown)..."
    fi

    if ! wget --show-progress -q -O "$tmp_file" "$maven_url"; then
        die "Failed to download Maven"
    fi

    log "INFO" "📦 Extracting Maven..."
    tar -xzf "$tmp_file" -C "$INSTALL_DIR" --strip-components=1 || die "Failed to extract Maven"
    rm -f "$tmp_file"
}

configure_mirror() {
    log "INFO" "🪞 Configuring Huawei Cloud mirror..."
    local maven_conf_dir="$INSTALL_DIR/conf"
    local maven_settings_file="$maven_conf_dir/settings.xml"
    local m2_dir="$HOME/.m2"
    local m2_settings_file="$m2_dir/settings.xml"

    mkdir -p "$m2_dir"

    cp "$maven_settings_file" "$m2_settings_file"
    log "INFO" "Copied settings.xml to ~/.m2/ (overwritten if exists)"

    if ! grep -q "huaweicloud" "$m2_settings_file"; then
        sed -i '/<\/mirrors>/i \
    <mirror>\
      <id>huaweicloud</id>\
      <mirrorOf>*</mirrorOf>\
      <url>https://repo.huaweicloud.com/repository/maven/</url>\
    </mirror>\
        ' "$m2_settings_file"
        log "INFO" "Added Huawei Cloud mirror to settings.xml"
    else
        log "INFO" "Huawei Cloud mirror already exists in settings.xml"
    fi
}

configure_environment() {
    log "INFO" "⚙️  Setting up environment variables..."

    if grep -q "MAVEN_HOME=" "$BASHRC_FILE"; then
        sed -i "s|^MAVEN_HOME=.*|MAVEN_HOME=${INSTALL_DIR}|" "$BASHRC_FILE"
    else
        echo "MAVEN_HOME=${INSTALL_DIR}" >> "$BASHRC_FILE"
    fi

    if grep -q "PATH=" "$BASHRC_FILE"; then
        if ! grep -q "\$MAVEN_HOME/bin" "$BASHRC_FILE"; then
            sed -i "s|^PATH=\(.*\)|PATH=\1:\$MAVEN_HOME/bin|" "$BASHRC_FILE"
        fi
    else
        echo 'PATH=$PATH:$MAVEN_HOME/bin' >> "$BASHRC_FILE"
    fi
}

main() {
    check_dependencies
    log "INFO" "🚀 Starting Maven installation..."
    install_maven
    configure_mirror
    configure_environment
    log "INFO" "🎉 Maven installation completed!"
}

main
```

#### gradle

```
#!/bin/bash
set -Eeuo pipefail

GRADLE_VERSION=""  # 
INSTALL_DIR="$HOME/.local/share/gradle"
BASHRC_FILE="$HOME/.bashrc"

LOG_LEVEL="INFO"
LOG_COLOR_INFO='\033[1;32m'
LOG_COLOR_WARN='\033[1;33m'
LOG_COLOR_ERROR='\033[1;31m'
LOG_COLOR_DEBUG='\033[1;36m'
LOG_COLOR_RESET='\033[0m'

log() {
    local level=$1
    local message=$2
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local color=""
    case $level in
        INFO) color=${LOG_COLOR_INFO} ;;
        WARN) color=${LOG_COLOR_WARN} ;;
        ERROR) color=${LOG_COLOR_ERROR} ;;
        DEBUG) color=${LOG_COLOR_DEBUG} ;;
        *) color=${LOG_COLOR_RESET} ;;
    esac
    if [[ $level == "DEBUG" && $LOG_LEVEL != "DEBUG" ]]; then
        return
    fi
    echo -e "${color}[${timestamp}] [${level}] ${message}${LOG_COLOR_RESET}" >&2
}

die() {
    local message=$1
    local code=${2-1}
    log "ERROR" "💥 ${message}"
    exit "$code"
}

check_dependencies() {
    local dependencies=("curl" "wget" "unzip")
    local missing_deps=()

    for dep in "${dependencies[@]}"; do
        if ! command -v "$dep" &>/dev/null; then
            missing_deps+=("$dep")
        fi
    done

    if [[ ${#missing_deps[@]} -gt 0 ]]; then
        log "WARN" "The following required tools are not installed: ${missing_deps[*]}"
        log "INFO" "Please install them and try again."
        exit 1
    fi

    log "INFO" "👍 All required tools are installed."
}

get_latest_gradle_file() {
    log "INFO" "Finding latest Gradle version..."
    local gradle_file=$(curl -sSL "https://mirrors.nju.edu.cn/gradle/?C=M&O=D" 2>/dev/null | grep -oP 'title="[^"]*bin\.zip"' | sed -E 's/title="|"$//g' | head -n 1)
    if [[ -z "$gradle_file" ]]; then
        die "No Gradle files found."
    fi
    echo "$gradle_file"
}

get_file_size() {
    local filename=$1
    local file_size=$(curl -sI "https://mirrors.nju.edu.cn/gradle/${filename}" | grep -i 'Content-Length' | awk '{print $2}' | tr -d '\r')
    if [[ -n "$file_size" ]]; then
        echo "$file_size"
    else
        echo "unknown"
    fi
}

install_gradle() {
    local gradle_file
    if [[ -n "$GRADLE_VERSION" ]]; then
        gradle_file="gradle-${GRADLE_VERSION}-bin.zip"
    else
        gradle_file=$(get_latest_gradle_file)
    fi

    local gradle_url="https://mirrors.nju.edu.cn/gradle/${gradle_file}"
    local gradle_zip="/tmp/${gradle_file}"

    local file_size=$(get_file_size "$gradle_file")
    if [[ "$file_size" != "unknown" ]]; then
        file_size=$(numfmt --to=iec --suffix=B --format="%.2f" "$file_size")
        log "INFO" "Downloading ${gradle_file} (${file_size})..."
    else
        log "WARN" "Downloading ${gradle_file} (file size unknown)..."
    fi

    if ! wget --show-progress -q -O "$gradle_zip" "$gradle_url"; then
        die "Failed to download Gradle"
    fi

    log "INFO" "Cleaning up $INSTALL_DIR..."
    rm -rf "$INSTALL_DIR"/*

    log "INFO" "Extracting Gradle..."
    mkdir -p "$INSTALL_DIR"
    unzip -q "$gradle_zip" -d "$INSTALL_DIR" || die "Failed to extract Gradle"
    rm -f "$gradle_zip"

    mv "$INSTALL_DIR"/gradle-*/* "$INSTALL_DIR"
    rm -rf "$INSTALL_DIR"/gradle-*
}

configure_mirror() {
    log "INFO" "Configuring Gradle mirror..."
    local gradle_init_dir="$HOME/.gradle"
    local gradle_init_file="$gradle_init_dir/init.gradle.kts"

    mkdir -p "$gradle_init_dir"

    cat <<EOF > "$gradle_init_file"
allprojects {
    repositories {
        maven{ url = uri("https://repo.huaweicloud.com/repository/maven");}
    }
    buildscript {
        repositories {
            maven{ url = uri("https://repo.huaweicloud.com/repository/maven");}
        }
    }
}
EOF

    log "INFO" "✅ Gradle mirror configured successfully."
}

configure_environment() {
    log "INFO" "Configuring Gradle environment variables..."
    local gradle_home="$INSTALL_DIR"

    if grep -q "GRADLE_HOME=" "$BASHRC_FILE"; then
        sed -i "/GRADLE_HOME=/c\GRADLE_HOME=$gradle_home" "$BASHRC_FILE"
    else
        echo "GRADLE_HOME=$gradle_home" >> "$BASHRC_FILE"
    fi

    if grep -q "PATH=" "$BASHRC_FILE"; then
        if ! grep -q "\$GRADLE_HOME/bin" "$BASHRC_FILE"; then
            sed -i "s|PATH=\(.*\)|PATH=\1:\$GRADLE_HOME/bin|" "$BASHRC_FILE"
        fi
    else
        echo 'PATH=$PATH:$GRADLE_HOME/bin' >> "$BASHRC_FILE"
    fi

    log "INFO" "Gradle environment variables configured."
}

main() {
    log "INFO" "👶 Starting Gradle installation..."
    check_dependencies
    install_gradle
    configure_mirror
    configure_environment
    log "INFO" "✅ Gradle installation completed successfully!"
}

main
```

#### docker

```
#!/bin/bash
set -Eeuo pipefail

LOG_LEVEL="INFO"
LOG_COLOR_INFO='\033[1;32m'
LOG_COLOR_WARN='\033[1;33m'
LOG_COLOR_ERROR='\033[1;31m'
LOG_COLOR_DEBUG='\033[1;36m'
LOG_COLOR_RESET='\033[0m'

log() {
    local level=$1
    local message=$2
    local timestamp=$(date +"%Y-%m-%d %H:%M:%S")
    local color=""
    case $level in
        INFO) color=${LOG_COLOR_INFO} ;;
        WARN) color=${LOG_COLOR_WARN} ;;
        ERROR) color=${LOG_COLOR_ERROR} ;;
        DEBUG) color=${LOG_COLOR_DEBUG} ;;
        *) color=${LOG_COLOR_RESET} ;;
    esac
    if [[ $level == "DEBUG" && $LOG_LEVEL != "DEBUG" ]]; then
        return
    fi
    echo -e "${color}[${timestamp}] [${level}] ${message}${LOG_COLOR_RESET}" >&2
}

die() {
    local message=$1
    local code=${2-1}
    log "ERROR" "💥 ${message}"
    exit "$code"
}

remove_old_docker() {
    log "INFO" "🧹 Removing old Docker versions..."
    sudo apt remove -y docker docker-engine docker.io containerd runc &>/dev/null || log "WARN" "No old Docker versions found."
    sudo apt autoremove -y &>/dev/null || log "WARN" "Failed to autoremove unused packages."
    log "INFO" "✅ Old Docker versions removed."
}

add_docker_gpg_key() {
    log "INFO" "🔑 Adding Docker GPG key..."
    curl -fsSL https://repo.huaweicloud.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg || die "Failed to add Docker GPG key."
    log "INFO" "✅ Docker GPG key added."
}

add_docker_repo() {
    log "INFO" "📦 Adding Docker repository (deb822 format)..."
    echo "Types: deb
URIs: https://repo.huaweicloud.com/docker-ce/linux/ubuntu
Suites: $(lsb_release -cs)
Components: stable
Signed-By: /etc/apt/keyrings/docker.gpg" | sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null || die "Failed to add Docker repository."
    log "INFO" "✅ Docker repository added."
}

install_docker() {
    log "INFO" "📥 Updating package index..."
    sudo apt update -y &>/dev/null || die "Failed to update package index."

    log "INFO" "📦 Installing Docker..."
    sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin &>/dev/null || die "Failed to install Docker."
    log "INFO" "✅ Docker installed successfully."
}

configure_docker_mirror() {
    log "INFO" "🪞 Configuring Docker mirror..."

    local mirrors=(
        "https://docker.1ms.run"
        "https://docker.m.daocloud.io"
        "https://docker.ketches.cn"
        "https://hub1.nat.tf"
        "https://hub2.nat.tf"
        "https://docker.1panel.live"
        "https://hub.rat.dev"
        "https://docker.amingg.com"
        "https://0bcac64191000fc80f47c017b8abbb60.mirror.swr.myhuaweicloud.com"
    )

    local selected_mirror=${mirrors[$RANDOM % ${#mirrors[@]}]}
    log "INFO" "🔧 Selected mirror: ${selected_mirror}"

    echo "{
  \"registry-mirrors\": [\"${selected_mirror}\"]
}" | sudo tee /etc/docker/daemon.json > /dev/null || die "Failed to configure Docker mirror."

    log "INFO" "✅ Docker mirror configured."
}

configure_user_permissions() {
    log "INFO" "👤 Configuring user permissions..."
    sudo usermod -aG docker "$(whoami)" || die "Failed to add user to Docker group."
    newgrp docker || log "WARN" "Failed to reload user groups. Please restart your session."
    log "INFO" "✅ User permissions configured."
}

restart_docker_service() {
    log "INFO" "🔄 Restarting Docker service..."
    sudo systemctl restart docker || die "Failed to restart Docker service."
    log "INFO" "✅ Docker service restarted."
}

main() {
    log "INFO" "👶 Starting Docker installation..."
    remove_old_docker
    add_docker_gpg_key
    add_docker_repo
    install_docker
    configure_docker_mirror
    configure_user_permissions
    restart_docker_service
    log "INFO" "✅ Docker installation completed successfully!"
}

main
```

