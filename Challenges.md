# Challenges 

### 1. Two Lock Files, Two Different Dependency Trees

**Problem:** The repo had both `package-lock.json` and `yarn.lock` with different package versions, so fixing one didn't fix the other.

**Solution:** Kept only `yarn.lock`, because the Dockerfile builds with Yarn. Added `--frozen-lockfile`, so the build fails if `yarn.lock` is out of date.

**Result:** One list of packages, and the scan checks exactly what gets built.

---

### 2. Upgrading All Packages Broke the Build

**Problem:** A full `yarn upgrade` broke the TypeScript build, because some new versions didn't work with the old code.

**Solution:** Updated only the vulnerable packages. For packages pulled in by other packages, used `resolutions` in `package.json` to force a safe version.

**Result:** The build works, and all high and critical findings are gone.

---

### 3. Dependency-Check Flagged Build Tools

**Problem:** Dependency-Check still failed because of build tools, which are only needed to build the app and aren't in the final image.

**Solution:** Skipped dev dependencies in the scan, because they never reach production.

**Result:** The gate checks only what actually runs, and the pipeline is green.

---

### 4. The Quality Gate Step Never Finished

**Problem:** The pipeline waited forever at the Quality Gate, because SonarQube had no way to send the result back to Jenkins.

**Solution:** Added a webhook in SonarQube that points to Jenkins (`/sonarqube-webhook/`), because Jenkins waits for SonarQube to report the result.

**Result:** The Quality Gate result arrives within seconds.

---

### 5. Dependency-Check Needed an NVD API Key

**Problem:** Dependency-Check couldn't download its vulnerability database, because the NVD now limits downloads without an API key.

**Solution:** Requested a free NVD API key and stored it in Jenkins as a Secret text credential (`nvd-api-key`).

**Result:** The database downloads and the scan runs.

---

### 6. Prometheus 3 Wouldn't Start

**Problem:** Prometheus failed to start with the tutorial's service file, because it points to the `consoles` folders, which were removed in Prometheus 3.

**Solution:** Removed the two console options from the service file.

**Result:** Prometheus starts and collects data.

---

### 7. Upgrading to Node 26

**Problem:** After switching to Node 26, the Docker build failed because `node:26-alpine` doesn't include Yarn anymore. Jenkins's Node tool also failed, because Node 26 needs the `libatomic1` library, which wasn't on the server.

**Solution:** Added `npm install -g yarn` to the Dockerfile and installed `libatomic1` on the Jenkins server.

**Result:** The app builds on Node 26 LTS.

---

### 8. The Browser Showed the Old Version

**Problem:** After a new deployment, the app still looked old, because the browser kept the old JavaScript files in its cache.

**Solution:** A hard reload (`Ctrl + Shift + R`) loads the new files.

**Result:** The new version shows right away.