# Bitrate in Chunk Names Configuration

This guide explains how to configure nginx-vod-module to include bitrate information in chunk names for both HLS and DASH streaming formats in mapping mode.

## Overview

By default, nginx-vod-module generates chunk names without bitrate information:
- **HLS**: `seg-1-v1-a1.ts`, `seg-2-v1-a1.ts`, etc.
- **DASH**: `fragment-1-v1.m4s`, `fragment-2-v1.m4s`, etc.

With the new configuration options, you can include bitrate in the chunk names:
- **HLS**: `seg-1-b900000-v1-a1.ts`, `seg-2-b900000-v1-a1.ts`, etc.
- **DASH**: `fragment-1-b900000-v1.m4s`, `fragment-2-b900000-v1.m4s`, etc.

## Configuration

### HLS Configuration

Add the following directive to your nginx configuration:

```nginx
location /hls/ {
    vod hls;
    vod_mode mapped;
    
    # Enable bitrate in HLS segment names
    vod_hls_include_bitrate_in_names on;
    
    # Optional: customize segment prefix (default: "seg")
    vod_hls_segment_file_name_prefix myseg;
    
    # Your other HLS configurations...
}
```

### DASH Configuration

Add the following directive to your nginx configuration:

```nginx
location /dash/ {
    vod dash;
    vod_mode mapped;
    
    # Enable bitrate in DASH fragment names
    vod_dash_include_bitrate_in_names on;
    
    # Optional: customize fragment prefix (default: "fragment")
    vod_dash_fragment_file_name_prefix myfrag;
    
    # Your other DASH configurations...
}
```

### Complete Example Configuration

```nginx
http {
    upstream mapping {
        server 127.0.0.1:3000;
    }

    server {
        listen 80;
        server_name example.com;

        # Common VOD settings
        vod_mode mapped;
        vod_upstream_location /mapping;
        vod_upstream_extra_args "token=$arg_token";

        # HLS with bitrate in names
        location ~ ^/hls/(.+)\.m3u8$ {
            vod hls;
            vod_hls_include_bitrate_in_names on;
            
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
            add_header Access-Control-Allow-Origin '*';
            expires 100d;
        }

        location ~ ^/hls/(.+)\.ts$ {
            vod hls;
            vod_hls_include_bitrate_in_names on;
            
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
            add_header Access-Control-Allow-Origin '*';
            expires 100d;
        }

        # DASH with bitrate in names
        location ~ ^/dash/(.+)\.mpd$ {
            vod dash;
            vod_dash_include_bitrate_in_names on;
            
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
            add_header Access-Control-Allow-Origin '*';
            expires 100d;
        }

        location ~ ^/dash/(.+)\.m4s$ {
            vod dash;
            vod_dash_include_bitrate_in_names on;
            
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Expose-Headers 'Server,range,Content-Length,Content-Range';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
            add_header Access-Control-Allow-Origin '*';
            expires 100d;
        }

        # Mapping endpoint
        location /mapping {
            proxy_pass http://mapping;
            proxy_set_header Host $host;
        }
    }
}
```

## Mapping Response Requirements

When using mapping mode with bitrate in chunk names, ensure your mapping endpoint returns media set JSON with proper bitrate information:

```json
{
  "sequences": [
    {
      "clips": [
        {
          "type": "source",
          "path": "/path/to/video.mp4"
        }
      ],
      "bitrate": {
        "v": 900000,
        "a": 64000
      }
    }
  ]
}
```

## Expected Results

### HLS Manifest (.m3u8)
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10.0,
seg-1-b900000-v1-a1.ts
#EXTINF:10.0,
seg-2-b900000-v1-a1.ts
#EXTINF:10.0,
seg-3-b900000-v1-a1.ts
#EXT-X-ENDLIST
```

### DASH Manifest (.mpd)
```xml
<SegmentTemplate
    timescale="1000"
    media="fragment-$Number$-b900000-$RepresentationID$.m4s"
    initialization="init-$RepresentationID$.mp4"
    duration="10000"
    startNumber="1">
</SegmentTemplate>
```

## Configuration Directives Reference

### `vod_hls_include_bitrate_in_names`
- **Syntax**: `vod_hls_include_bitrate_in_names on | off`
- **Default**: `off`
- **Context**: `http`, `server`, `location`
- **Description**: When enabled, includes the bitrate in HLS segment file names

### `vod_dash_include_bitrate_in_names`
- **Syntax**: `vod_dash_include_bitrate_in_names on | off`
- **Default**: `off`
- **Context**: `http`, `server`, `location`
- **Description**: When enabled, includes the bitrate in DASH fragment file names

## Notes

1. **Bitrate Source**: The bitrate value is taken from the `media_info.bitrate` field of the first track in the media set.

2. **Format**: Bitrate is included as `-b{bitrate}` in the filename (e.g., `-b900000` for 900 kbps).

3. **Zero Bitrate**: If bitrate is 0 or not available, the bitrate portion is omitted from the filename.

4. **Backward Compatibility**: These are optional features. Existing configurations will continue to work without modification.

5. **Performance**: The bitrate lookup adds minimal overhead to the manifest generation process.

## Troubleshooting

### Bitrate Not Appearing in Names
1. Verify the configuration directive is enabled
2. Check that the mapping response includes bitrate information
3. Ensure the media file has bitrate metadata
4. Check nginx error logs for any configuration errors

### Invalid Segment URLs
1. Verify URL parsing logic accounts for the new naming format
2. Update any custom segment parsing in your application
3. Test with a simple configuration first

This configuration enhancement allows for more descriptive chunk names that can be useful for debugging, caching strategies, and analytics.