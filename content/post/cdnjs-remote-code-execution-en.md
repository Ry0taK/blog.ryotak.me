---
title: "Remote code execution in cdnjs of Cloudflare"
date: 2021-07-01T12:53:03+09:00
draft: false
tags: ["cdnjs", "Vulnerability", "Go", "Supply Chain", "RCE"]
---

## Preface

Cloudflare, which runs cdnjs, is running a "Vulnerability Disclosure Program" on HackerOne, which allows hackers to perform the vulnerability assessment.  
This article describes a vulnerability reported through this program, and published with the permission of the Cloudflare security team. So this aritcle is not intended to recommend you to perform an unauthorized vulnerability assessment.  
If you found any vulnerabilities in Cloudflare's product, please report it to [Cloudflare's vulnerability disclosure program](https://hackerone.com/cloudflare).

## TL;DR
There was a vulnerability in the cdnjs library update server that could execute arbitrary commands, and as a result, cdnjs could be completely compromised.  
This allows an attacker to tamper 12.6%[^1] of all websites on the internet.  

[^1]: Quoted from [W3Techs](https://w3techs.com/technologies/details/cd-cdnjs) as of 2 July 2021. Due to the presence of SRI, fewer websites could be tampered immediately.  

## About cdnjs

[cdnjs](https://cdnjs.com) is a JavaScript/CSS library CDN that is owned by Cloudflare, which is used by 12.6% of all websites on the internet as of 2 July 2021.  
This is the second most widely used library CDN after 12.8%[^2] of [Google Hosted Libraries](https://developers.google.com/speed/libraries), and considering the current usage rate, it will be the most used JavaScript library CDN in the near future.  

<div style="text-align: center;">
  <img src="/img/cdnjs-usage.png" alt="Usage rate of cdnjs on the internet">
  <p>Usage graph of cdnjs from <a href="https://w3techs.com/technologies/details/cd-cdnjs" target="_blank">W3Techs</a>, as of 2 July 2021</p>
</div>

[^2]: Quoted from [W3Techs](https://w3techs.com/technologies/details/cd-googlelibraries) as of 2 July 2021.

## Reason for investigation

A few weeks before my last investigation into "[Remote code execution in Homebrew by compromising the official Cask repository](http://localhost:19191/post/homebrew-security-incident-en/)", I was investigating supply chain attacks.  
While finding a service that many software depends on, and is allowing users to perform the vulnerability assessment, I found cdnjs. So I decided to investigate it.  

## Initial investigation

While browsing the cdnjs website, I found the following description.  

> Couldn't find the library you're looking for?  
You can make a request to have it added on our GitHub repository.

I found out that the library information is managed on the GitHub repository, so I checked repositories of the GitHub Organization that is used by cdnjs.  

As a result, it was found that the repository is used in the following ways.

- [cdnjs/packages](https://github.com/cdnjs/packages): Stores library information that is supported in cdnjs
- [cdnjs/cdnjs](https://github.com/cdnjs/cdnjs): Stores files of libraries
- [cdnjs/logs](https://github.com/cdnjs/logs): Stores update logs of libraries
- [cdnjs/SRIs](https://github.com/cdnjs/SRIs): Stores SRI (Subresource Integrity) of libraries
- [cdnjs/static-website](https://github.com/cdnjs/static-website): Source code of cdnjs.com
- [cdnjs/origin-worker](https://github.com/cdnjs/origin-worker): Cloudflare Worker for origin of cdnjs.cloudflare.com
- [cdnjs/tools](https://github.com/cdnjs/tools): cdnjs management tools
- [cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible): Ansible repository of the cdnjs library update server

As you can see from these repositories, most of the cdnjs infrastructure is centralized in this GitHub Organization.  
I was interested in [cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible) and [cdnjs/tools](https://github.com/cdnjs/tools) because it automates library updates.  
After reading codes of these 2 repositories, it turned out [cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible) executes `autoupdate` command of [cdnjs/tools](https://github.com/cdnjs/tools/tree/b6833e08108b2a06b3b3e5f212d604b5951ff924) in the cdnjs library update server periodically, to check updates of library from [cdnjs/packages](https://github.com/cdnjs/packages) by downloading npm package / Git repository of libraries.  

## Investigation of automatic update

The automatic update function updates the library by downloading the user-managed Git repository / npm repository and copying the target file from them.  
npm registry compress libraries into `.tgz` to make it downloadable.  
Since the tool for this automatic update is written in Go, I guessed that it may use Go's `compress/gzip` and `archive/tar` to extract archive.  
Go's `archive/tar` returns the file path contained in the archive without sanitizing[^3], so if the archive is extracted into the disk based on the filename returned from `archive/tar`, archives that contains file path like `../../../../../../../tmp/test` may overwrite arbitrary files on the system.  
From the information in [cdnjs/bot-ansible](https://github.com/cdnjs/bot-ansible), I knew that some scripts were running regularly and the user that runs `autoupdate` command had write permission for them, so I focused on overwriting files via path traversal.  

[^3]: https://github.com/golang/go/issues/25849

![Image of crafted tgz file to perform path traversal](/img/cdnjs-tgz-slip.png)

## Path traversal
To find path traversal, I started reading `main` function of `autoupdate` command.  
```go
func main() {
        [...]	
		switch *pckg.Autoupdate.Source {
		case "npm":
			{
				util.Debugf(ctx, "running npm update")
				newVersionsToCommit, allVersions = updateNpm(ctx, pckg)
			}
		case "git":
			{
				util.Debugf(ctx, "running git update")
				newVersionsToCommit, allVersions = updateGit(ctx, pckg)
			}
		[...]
}
```

As you can see from the code snippet above, if `npm` is specified as a source of auto update, it passes package information to `updateNpm` function.  

```go
func updateNpm(ctx context.Context, pckg *packages.Package) ([]newVersionToCommit, []version) {
		[...]
		newVersionsToCommit = doUpdateNpm(ctx, pckg, newNpmVersions)
        [...]
}
```

Then, `updateNpm` passes information about the new library version to `doUpdateNpm` function.  

```go
func doUpdateNpm(ctx context.Context, pckg *packages.Package, versions []npm.Version) []newVersionToCommit {
    [...]
	for _, version := range versions {
        [...]
		tarballDir := npm.DownloadTar(ctx, version.Tarball)
		filesToCopy := pckg.NpmFilesFrom(tarballDir)
        [...]
}
```
And `doUpdateNpm` passes the URL of `.tgz` file into `npm.DownloadTar`.    

```go
func DownloadTar(ctx context.Context, url string) string {
	dest, err := ioutil.TempDir("", "npmtarball")
	util.Check(err)

	util.Debugf(ctx, "download %s in %s", url, dest)

	resp, err := http.Get(url)
	util.Check(err)

	defer resp.Body.Close()

	util.Check(Untar(dest, resp.Body))
	return dest
}
```
Finally, pass the `.tgz` file obtained using `http.Get` to the `Untar` function.  

```go
func Untar(dst string, r io.Reader) error {
	gzr, err := gzip.NewReader(r)
	if err != nil {
		return err
	}
	defer gzr.Close()
	tr := tar.NewReader(gzr)
	for {
		header, err := tr.Next()
        [...]
		// the target location where the dir/file should be created
		target := filepath.Join(dst, removePackageDir(header.Name))
        [...]
		// check the file type
		switch header.Typeflag {
        [...]
		// if it's a file create it
		case tar.TypeReg:
			{
                [...]
				f, err := os.OpenFile(target, os.O_CREATE|os.O_RDWR, os.FileMode(header.Mode))
                [...]
				// copy over contents
				if _, err := io.Copy(f, tr); err != nil {
					return err
				}
			}
		}
	}
}
```

As I guessed, `compress/gzip` and `archive/tar` were used in `Untar` function to extract `.tgz` file.  
At first I thought that it's sanitizing the path in `removePackageDir` function, but when I checked the contents of the function, I noticed that it's just removing the string `package/` from the path.  
From these code snippets, I confirmed that arbitrary code can be executed after performing path traversal from the `.tgz` file published to npm and overwriting the script that is executed regularly on the server.  


## Demonstration of vulnerability
Because Cloudflare is running a vulnerability disclosure program on HackerOne, it's likely that HackerOne's triage team won't forward the report to Cloudflare unless it indicates that the vulnerability is actually exploitable.    
Therefore, I decided to do a demonstration to show that the vulnerability can actually be exploited.  

The attack procedure is as follows.
1. Publish `.tgz` file that contains crafted filename to the npm registry.
2. Wait for cdnjs library update server to process the crafted `.tgz` file.
3. The contents of the file that is published in step 1 are written into a regularly executed script file and arbitrary command is executed.

... and then, for some reasons, I started wondering how automatic updates based on the Git repository works.  
So, I read codes a bit before demonstrating the vulnerability, and it seemed that the symlinks aren't considered when copying files from the Git repository.  
As Git supports symbolic links by default, it may be possible to read arbitrary files from cdnjs library update server by adding symlink into the Git repository.  

If the regularly executed script file is overwritten to execute arbitrary command, the automatic update function may be broken, so I decided to check the arbitrary file reading first.    
	Along with this, the attack procedure was changed as follows.  
1. Add a symbolic link that points harmless file (Assumed `/proc/self/maps` here) into the Git repository.
2. Publish new version in the repository.
2. Wait for cdnjs library update server to process the crafted repository.
4. The specified file is published on cdnjs.

It was around 20:00 at this point, but what I have to do was creating symlink, so I decided to eat dinner after making the symbolic link and publishing it.[^4]  
```bash
ln -s /proc/self/maps test.js
```

[^4]: I don't know if this is correct, but I remember that the dinner on that day was frozen gyoza (dumplings). (It was yummy!)  

## Incident
Once I finished the dinner and returning to my PC desk, I was able to confirm that cdnjs has released a version containing symbolic links.    
After checking the contents of the file to send report, I was surprised.  
Surprisingly, clearly sensitive information such as `GITHUB_REPO_API_KEY` and `WORKERS_KV_API_TOKEN` was displayed.  
I couldn't understand what happened for a moment, and when I checked the command log, I found that I accidentally put a link to `/proc/self/environ` instead of `/proc/self/maps`.[^5]  
As mentioned earlier, if cdnjs' GitHub Organization is compromised, it's possible to compromise most of cdnjs infrastructure.  
I needed to take immediate action, so I sent the report that only contains link that shows the current situation, and requested them to revoke all credentails.  

[^5]: Because I was tired from work and I was hungry, I ran the command completed by shell without any confirmation.

At this point I was very confused and hadn't confirmed it, but in fact these tokens were invalidated before I sent the report.  
It seems that GitHub automatically notified immediately because `GITHUB_REPO_API_KEY` (API key of GitHub) was included in the repository, and Cloudflare started incident response immediatialy after the notification.  
I felt that they're great security team because they invalidated all credentials within minutes after cdnjs processed the specially crafted repository.  

## Post processing
After the incident, I investigated what could be impacted.  
`GITHUB_REPO_API_KEY` was an API key for [robocdnjs](https://github.com/robocdnjs), which belongs to [cdnjs](https://github.com/cdnjs) organization, and had write permission against each repositories.  
Which means it was possible to tamper arbitrary libraries on the cdnjs, or tamper the cdnjs.com itself.  
Also, `WORKERS_KV_API_TOKEN` had a permission against KV of Cloudflare Workers that is used in the cdnjs, it could be used to tamper the libraries on KV cache, too.  
By combining these permissions, the core part of cdnjs, such as the origin data of cdnjs, the KV cache, and even the cdnjs website, could be completely tampered.    

## Conclusion
In this article, I described the vulnerability that was existed in cdnjs.  
While this vulnerability could be exploited without any special skills, it could impact many websites.  
Considering that there are many vulnerabilities in the world, such as this one, which are easy to exploit but have a very large impact, I feel that it's very scary.  
If you have any questions/comments about this article, please send a message to ([@ryotkak](https://twitter.com/ryotkak)) on Twitter.  
## Timeline
|Date (JST)|Event|
|----|----|
|April 6, 2021 19:00|Found a vulnerability|
|April 6, 2021 20:00|Published a crafted symlink|
|April 6, 2021 20:30|cdnjs processed the file|
|At the same time|GitHub sent an alert to Cloudflare|
|At the same time|Cloudflare started an incident response|
|Within minutes|Cloudflare finished revokation of credentials|
|April 6, 2021 20:40|I sent an initial report|
|April 6, 2021 21:00|I sent detailed report|
|April 7, 2021-|Secondary fix has applied |
|June 3, 2021|Complete fix has applied|
|July XX, 2021|Published this article|