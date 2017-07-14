# README Formatting

Sub-Heading
---

## Sub-Heading
First paragraph

Second paragraph

---

**Bold Text**

*Italic Text*

---

## Blocks of Code
Surround commands or blocks of code with backticks

For a command, use one back tick in the beginning and end of the command:
`command.line`

For a block of code, put three above the block of code dnd three a the end:
```
#!/usr/bin/env groovy
@Library('pipeline')
import net.zonarsystems.pipeline.ApplicationPipeline
```

---

**Bullet Points**  

  * First bullet
  * Second bullet
  * Third bullet
  
**Numbered Lists**

  1. First
  2. Second
  3. Third

**Inserting Links**
[text to link here](http://www.website.com)

<p>Creating in Confluence to edit with engineer (as alternative to GitHub) for writing the first draft b/c it's cumbersome in GitHub.</p>
<h1>dr-ect</h1>
<p>This project contains the Docker and Kubernetes <ac:inline-comment-marker ac:ref="bfd69b10-52c2-47ca-81b3-b98a1acb7d0f">definitions</ac:inline-comment-marker> required to deploy the EVIR Configuration Тool (ECT) to a disaster-recovery Kubernetes cluster.</p>
<h2>Requirements</h2>
<p>All of the prod/dev disaster-recovery databases must be up and running with correct cluster DNS names set up (i.e., dev-db-001, misc-db etc.).</p>
<h2>Usage</h2>
<p>All DR Helm charts are installed via the Zonar DR pipeline. To install and run manually, clone this repo and install via:</p>
<pre>  <code>helm install charts/ect --name dr-ect --namespace prod --set githubUsername=REDACTED --set githubPassword=REDACTED  --set applicationEnv=production</code>
</pre>
<h2>Configuration</h2>
<p>ECT configuration files (app.ini, ,.htaccess and httpd.conf) have been converted into a Kubernetes config map, templated with values from <a href="https://confluence.zonarsystems.net/charts/ect/values.yaml">values.yaml</a>.</p>
<h3>Configuration settings</h3>
<table class="wrapped">
  <colgroup>
    <col/>
    <col/>
    <col/>
  </colgroup>
  <tbody>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Default</th>
    </tr>
    <tr>
      <td>
        <pre>ectRepo</pre>
      </td>
      <td>Github repository with production-quality ECT code in master branch</td>
      <td>
        <code>
          <span class="nolink">https://github.com/ZonarSystems/ect.git</span>
        </code>
      </td>
    </tr>
    <tr>
      <td>
        <br/>
      </td>
      <td>
        <br/>
      </td>
      <td>
        <br/>
      </td>
    </tr>
    <tr>
      <td>
        <br/>
      </td>
      <td>
        <br/>
      </td>
      <td>
        <br/>
      </td>
    </tr>
  </tbody>
</table>
<p>Maintained by <a href="mailto:maratoid@gmail.com">Marat Garafutdinov</a> at Samsung SDS</p>
