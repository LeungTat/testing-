```
import React, { useState, useEffect, useCallback } from "react";
import { AccountRaw } from "../../../../../types";
import invokeApi from "../../../../../libs/apiGateway";
import { inflate } from "pako";
import RefreshIcon from "../../../../../lib/icons/RefreshIcon";
import _ from "lodash";

interface ResourceDescription {
  ResourceType: string;
  FriendlyName: string;
  NameKey: string;
  ValueKey?: string;
  DefaultValue?: string;
}

const resourceDisplayItems: Record<string, ResourceDescription> = {
  sidecar_proxy_whitelist: {
    ResourceType: "sidecar_proxy_whitelist",
    FriendlyName: "Sidecar Proxy Whitelist",
    NameKey: "Id",
  },
};

type ResourceType = keyof typeof resourceDisplayItems;
interface ResourceCacheDict {
  [index: string]: string | ResourceCacheDict;
}

interface CompressedResourceCache {
  compressed: boolean;
  data: string;
}

interface ResourcesProps {
  account: AccountRaw;
}

function base64ToBuffer(base64: string): Uint8Array {
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes;
}

const decompress = (compressedData: string): Array<ResourceCacheDict> => {
  return JSON.parse(new TextDecoder().decode(inflate(base64ToBuffer(compressedData))));
};

const SidebarProxyWhitelist: React.FC<ResourcesProps> = ({ account }) => {
  const [isLoading, setIsLoading] = useState<boolean>(false);
  const [resources, setResources] = useState<Array<ResourceCacheDict>>([]);

  const getResources = useCallback(async () => {
    setIsLoading(true);
    try {
      const maybeCompressedResources = await invokeApi<Array<ResourceCacheDict> | CompressedResourceCache>({
        path: "/resource_cache",
        queryParams: {
          account_id: account.id,
          resource_type: resourceDisplayItems["sidecar_proxy_whitelist"].ResourceType,
        },
      });

      let fetchedResources: Array<ResourceCacheDict>;
      if ("compressed" in maybeCompressedResources && maybeCompressedResources.compressed === true) {
        fetchedResources = decompress(maybeCompressedResources.data);
      } else {
        fetchedResources = maybeCompressedResources as Array<ResourceCacheDict>;
      }

      if (fetchedResources.length === 0) {
        fetchedResources = [];
      }

      setResources(fetchedResources);
    } catch (e) {
      console.log(e);
      alert("Unable to complete request. Access token may have expired.");
    }
    setIsLoading(false);
  }, [account.id]);

  useEffect(() => {
    getResources();
  }, [getResources]);

  const renderContent = () => {
    if (isLoading) {
      return <RefreshIcon size="medium" />;
    }
    if (resources.length > 0) {
      return (
        <div data-cy="resource_cache_output">
          {_.map(resources, (value, key) => (
            <div key={key}>{JSON.stringify(value)}</div>
          ))}
        </div>
      );
    }
    return <h4>No results</h4>;
  };

  return (
    <div>
      {renderContent()}
    </div>
  );
};

export default SidebarProxyWhitelist;




```
