```
import ipaddress

def clean(self):
    """Perform additional field validation"""
    if not self.account:
        raise ValidationError("Account is invalid!")
    account_class = int(self.account.account_type)

    all_existing_cidr_blocks = [ipaddress.ip_network(block) for block in CidrBlock.objects.all()]

    # only number 4 or above will be considered as production
    account_category = "prod" if account_class >= 4 else "non-prod"
    ok, result = get_cidr(
        is_cid=self.account.is_cid,
        cidr_blocks=self.VALID_CIDR_BLOCKS,
        region_code=self.region,
        environment=account_category,
    )
    if not ok:
        raise ValidationError(result)
    
    valid_cidr_blocks = result
    if not self.ipv4_subnet_address:
        for valid_cidr_block in valid_cidr_blocks:
            cidr_block_head = ipaddress.ip_network(valid_cidr_block)
            # Check if the CIDR block head is strictly within the valid CIDR range
            if not any(cidr_block_head.subnet_of(ipaddress.ip_network(valid_block)) for valid_block in valid_cidr_blocks):
                raise ValidationError("Invalid CIDR block - the CIDR block is not within the valid range.")
        if self.find_cidr_block(cidr_block_head, valid_cidr_block, all_existing_cidr_blocks):
            raise ValidationError("Cannot find CIDR block!")
    else:
        cidr_block_head = ipaddress.ip_network(f"{self.ipv4_subnet_address}/{self.ipv4_subnet_mask}")
        if not any(cidr_block_head.subnet_of(ipaddress.ip_network(valid_block)) for valid_block in valid_cidr_blocks):
            raise ValidationError("Invalid IPv4 subnet address - the CIDR has already been allocated")

def find_cidr_block(self, self_cidr_block_head, valid_cidr_block, all_existing_cidr_blocks):
    while not self.validate_exist_pk(self_cidr_block_head, valid_cidr_block, all_existing_cidr_blocks):
        try:
            list(ipaddress.ip_network(str(valid_cidr_block)).address_exclude(self_cidr_block_head))
            cidr_block_head = ipaddress.ip_network(str(self_cidr_block_head))
            if not any(cidr_block_head.subnet_of(ipaddress.ip_network(valid_block)) for valid_block in valid_cidr_blocks):
                raise ValidationError("Invalid IPv4 subnet address - the CIDR has already been allocated")
        except ValueError:
            return False
    return True

```
