# Description
I was working on a lambda function that generates new volumes, which was working fine. But when I run the Terraform plan, I saw that Terraform detects the new volumes and thus wanted to recreate the volumes so it aws resources can reflect the state.
# Solution (GO)
I had to update the TF file state in the same lambda function, I was replacing the old volumes IDs with the new one. This prevented Terraform from recreating volumes, but still I had the problem with the aws_volume_attachment as the ID was calculated by Terraform and I had nowhere to get the ID from. 
I came across a really intresting blog [https://heap.io/blog/engineering/terraform-gotchas]
which explian that the way Terraform calculate the ID of the aws_volume_attachment is as follow :

```go
func volumeAttachmentID(name, volumeID, instanceID string) string {
	var buf bytes.Buffer
	buf.WriteString(fmt.Sprintf("%s-", name))
	buf.WriteString(fmt.Sprintf("%s-", instanceID))
	buf.WriteString(fmt.Sprintf("%s-", volumeID))
	return fmt.Sprintf("vai-%d", hashcode.String(buf.String()))
}
```

here is the whole code in GO to calculate the aws_attachment_volume ID:

```go
package main

import (
	"bytes"
	"fmt"
	"hash/crc32"
)

func String(s string) int {
	v := int(crc32.ChecksumIEEE([]byte(s)))
	if v >= 0 {
		return v
	}
	if -v >= 0 {
		return -v
	}

	return 0
}

func main() {
   
   var buf bytes.Buffer
	// vai-3972684014
    buf.WriteString(fmt.Sprintf("%s-", "/dev/sdb"))
	buf.WriteString(fmt.Sprintf("%s-", "i-08254b53e779ab35c"))
	buf.WriteString(fmt.Sprintf("%s-", "vol-07d059ab6396a5652"))
	
	fmt.Printf("vai-%d", String(buf.String()))
}
```

so I had to calculate the ID in the lambda function using the new volumeID, the device name and the instanceID. and then replace the old aws_volume_attachment with the new one

# Soltuon (python)

```python
import binascii

vai = binascii.crc32(bytes('{}-{}-{}-'.format('/dev/sdb','i-08254b53e779ab35c','vol-07d059ab6396a5652'), 'utf-8'))
print('vai-%s'%vai)
=> vai-3972684014
```
