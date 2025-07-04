# SonarQube Scanner Hanging Fix

## Issue Description

The SonarQube scanner hangs during the "Load/download plugins" phase, causing the pipeline to timeout or run for an extremely long time.

## Symptoms

```
INFO: Loading required plugins
INFO: Load plugins index
INFO: Load plugins index (done) | time=144ms
INFO: Load/download plugins
[HANGS HERE - NO FURTHER OUTPUT]
```

## Root Causes

1. **Network/Connectivity Issues**: Slow or unstable connection to SonarCloud/SonarQube server
2. **Large Project Size**: Too many files being analyzed simultaneously
3. **Scanner Cache Issues**: Corrupted or large cache files
4. **Java Memory Issues**: Insufficient heap space for the scanner
5. **Rate Limiting**: SonarCloud throttling requests

## Solutions Applied

### 1. Clear Scanner Cache Before Analysis

```yaml
# Clear SonarQube cache to prevent hanging issues
- bash: |
    rm -rf ~/.sonar/cache || true
    echo "SonarQube cache cleared"
  displayName: "Clear SonarQube Cache"
  condition: and(succeeded(), ne(variables['SONAR_PROJECT_KEY'], ''))
```

### 2. Increase Timeout and Memory Settings

```yaml
- task: SonarQubePrepare@7
  inputs:
    # ... existing config ...
    extraProperties: |
      # ... existing properties ...
      sonar.scanner.responseTimeout=300
      sonar.scanner.socketTimeout=300
      sonar.scanner.javaOpts=-Xmx2g -XX:MaxMetaspaceSize=512m
  timeoutInMinutes: 10
```

### 3. Add More Aggressive File Exclusions

```properties
# Exclude more file types that don't need analysis
sonar.exclusions=**/bin/**/*,**/obj/**/*,**/Migrations/**/*,**/*.Designer.cs,**/ModelSnapshot.cs,**/Program.cs,**/tests/**/*,**/packages/**/*,**/*.min.js,**/*.min.css
```

### 4. Add Pre-Analysis Delay

```yaml
# Optional: Add delay before SonarQube analysis to help with rate limiting
- bash: |
    echo "Waiting before SonarQube analysis to prevent rate limiting..."
    sleep 15
  displayName: "Pre-Analysis Delay"
  condition: and(succeeded(), ne(variables['SONAR_PROJECT_KEY'], ''))
```

### 5. Set Analysis Timeout

```yaml
- task: SonarQubeAnalyze@7
  displayName: "Run SonarQube Analysis"
  condition: and(succeeded(), ne(variables['SONAR_PROJECT_KEY'], ''))
  timeoutInMinutes: 15
```

## Emergency Fallback: Minimal Configuration

If the hanging persists, temporarily use minimal SonarQube configuration to isolate the issue:

```yaml
- task: SonarQubePrepare@7
  inputs:
    SonarQube: "SonarQube"
    scannerMode: "dotnet"
    projectKey: "$(SONAR_PROJECT_KEY)"
    projectName: "$(SONAR_PROJECT_NAME)"
    # Remove all extraProperties temporarily
  timeoutInMinutes: 5
```

## Verification

After applying these fixes:

- [ ] Cache is cleared before each analysis
- [ ] Scanner has adequate memory allocation
- [ ] Timeout limits prevent infinite hanging
- [ ] File exclusions reduce analysis scope
- [ ] Pipeline completes within reasonable time (< 15 minutes for analysis)

## Files Modified

1. `build/azure-pipelines/ci-build-test.yml` - Added cache clearing, timeouts, memory settings
2. `sonar-project.properties` - Enhanced file exclusions
3. `build/SONARQUBE_COMPLETE_GUIDE.md` - Updated troubleshooting section

This fix should resolve the scanner hanging issue and ensure reliable SonarQube analysis execution.
